# Use oc-mirror v2 for an OpenShift Integrated Registry

![GitHub release (latest by date)](https://img.shields.io/github/v/release/kevchu3/oc-mirror-v2-integrated-registry?color=blue&style=plastic)
![GitHub](https://img.shields.io/github/license/kevchu3/oc-mirror-v2-integrated-registry?color=blue&style=plastic)

## Background

The following procedures are designed for an airgapped Red Hat OpenShift cluster to mirror images to the OpenShift integrated registry, to reduce setup time and eliminate network connectivity to an external registry.  Mirroring uses the oc-mirror OpenShift CLI (oc) plugin, which mirrors all required OpenShift Container Platform content and other images to your mirror registry.  We'll use the oc-mirror v2 plugin ([refer to the docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/disconnected_environments/mirroring-in-disconnected-environments#about-installing-oc-mirror-v2)) to mirror to the cluster's OpenShift integrated registry, typically available at `image-registry.openshift-image-registry.svc.cluster.local:5000`.

# Design

This design pattern assumes the cluster is not fully airgapped and there is limited connectivity to the internet.  In a fully airgapped scenario, since the integrated registry is responsible for providing infrastructure images, the unavailability of the registry during an upgrade is a risk as a potential point of cyclical failure.  In addition, in a fully airgapped scenario, additional images for the operator catalog will need to be mirrored, and `CatalogSource` configured for an offline registry.

An `ImageDigestMirrorSet` (IDMS) for the cluster will be configured to redirect images to the internal registry.  Since applying IDMS requires a node cordon and drain, we've opted to handcraft an IDMS manifest that will be consistent over time even if operators add images to their upstream repositories.  There is no guarantee that these upstream repositories (i.e. `registry.redhat.io/container-native-virtualization`) will not change paths over time, but it is infrequent at best.

Prior to mirroring images, the OpenShift integrated registry should be configured for persistent storage with ReadWriteMany access mode and two or more replicas to support high availability.

## Setup

The oc-mirror v2 binary will mirror images based on an `ImageSetConfiguration` to target repositories.  An example [imagesetconfiguration.yaml](manifests/imagesetconfiguration.yaml) has been provided and should be customized based on your mirroring needs.  I've opted to wrap my `ImageSetConfiguration` inside a `ConfigMap` to be mounted to my mirroring job pod.  To determine the operator packages to specify in the `ImageSetConfiguration`, you can download the `oc-mirror` binary and run the following command (replace the tag with your OpenShift version):
```
./oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.18
```

You can view the [Container Platform Update Graph](https://access.redhat.com/labs/ocpupgradegraph/update_path) and your cluster's OperatorHub to determine specific cluster and operator package target versions to download and update.  As a good practice, you should also include the source (current) cluster and operator package versions, so that if pods must be drained prior to upgrading, they will pull from the internal registry.  If you are locking operator packages to a specific version, you can run the following command to determine the minVersion and maxVersion to use:

```
./oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.18 --package=kubevirt-hyperconverged --channel=stable
```

The OpenShift integrated registry stores images in persistent storage (if configured) in the openshift-image-registry namespace, with references from imagestreams in other namespaces.  It follows a `<registry>/<repository>/<image>[:tag] or [@sha256:digest]` format with a restriction of two levels of nesting and thus oc-mirror must use `--max-nested-paths=2`.  The registry also assumes the repository already exists as an OpenShift namespace prior to pushing images to it.  Thus, the namespaces will need to be created before mirroring.  You can also use the [Red Hat Ecosystem Catalog](https://catalog.redhat.com/software/containers/explore) to determine the source registry and repository paths that will need to be mirrored, replacing the repository path with the OpenShift namespaces that will need to be created.

Based on the example [imagesetconfiguration.yaml](manifests/imagesetconfiguration.yaml), we will create the following namespaces and configure RBAC to allow [System users](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/understanding-authentication#rbac-users_understanding-authentication) to pull images.

```
# openshift namespace already exists
oc create ns openshift4                        # maps to registry.redhat.io/openshift4 for local-storage-operator and kubernetes-nmstate-operator
oc create ns redhat                            # maps to registry.redhat.io/redhat for redhat-operator-index but we won't fetch it airgapped
oc create ns container-native-virtualization   # maps to registry.redhat.io/container-native-virtualization for kubevirt-hyperconverged
oc create ns migration-toolkit-virtualization  # maps to registry.redhat.io/migration-toolkit-virtualization for kubevirt-hyperconverged

oc adm policy add-role-to-group system:image-puller system:unauthenticated -n openshift
oc adm policy add-role-to-group system:image-puller system:unauthenticated -n openshift4
oc adm policy add-role-to-group system:image-puller system:unauthenticated -n redhat
oc adm policy add-role-to-group system:image-puller system:unauthenticated -n container-native-virtualization
oc adm policy add-role-to-group system:image-puller system:unauthenticated -n migration-toolkit-virtualization
```

## Mirroring

Create the `oc-mirror` namespace, job, and related manifests:
```
oc create ns oc-mirror
oc apply -f manifests
```

In the event that your job pod fails, you can create a debug pod and rerun the oc-mirror commands from within the pod (the pod displays the inline commands but does not automatically execute them when running a debug pod).  Image layer blobs that have already been pushed to the registry will not need to be repushed, so the mirroring process should largely pick up where it left off.

```
$ oc get pods
NAME                  READY   STATUS    RESTARTS   AGE
run-oc-mirror-6x6hl   1/1     Running   0          33s
$ oc debug pod/run-oc-mirror-6x6hl
Starting pod/run-oc-mirror-6x6hl-debug-5ccp2, command was: /bin/bash -c # Download oc-mirror
cd /tmp
curl https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest/oc-mirror.tar.gz -o oc-mirror.tar.gz
tar -xzvf oc-mirror.tar.gz
chmod +x oc-mirror
# Login to registry
oc registry login --registry ${REGISTRY} --to=/.docker/destination-registry.json

# Downloads the Pull secret
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > /tmp/pull-secret.json

# Combine pull secret and registry auth
jq -s 'reduce .[] as $item ({}; .auths += $item.auths)' "/tmp/pull-secret.json" "/.docker/destination-registry.json" > /.docker/config.json

# Run oc-mirror
./oc-mirror --v2 --config=/tmp/imagesetconfiguration.yaml docker://${REGISTRY} --dest-tls-verify=${DEST_TLS_VERIFY} --max-nested-paths=2 --retry-times 10 --workspace=file:///opt/oc-mirror --cache-dir=/opt/oc-mirror --log-level debug

Pod IP: 10.130.1.85
If you don't see a command prompt, try pressing enter.
```

## Applying ImageDigestMirrorSet

After the content has been mirrored, the ImageDigestMirrorSet can be applied.  An example [imagedigestmirrorset.yaml](idms/imagedigestmirrorset.yaml) has been provided and should be customized.  Applying and updating this manifest will cordon and drain nodes, and I've customized the manifest to be resilient across OpenShift upgrades to reduce the cordon and drain activities.

```
oc apply -f idms/imagedigestmirrorset.yaml
```

## Upgrading

Finally, upgrade OpenShift.  Since we specified specific versions in the [imagesetconfiguration.yaml](manifests/imagesetconfiguration.yaml), remember to use the same versions.

To verify the images are being properly pulled through the OpenShift integrated registry, you can search the registry pod logs for corresponding image pulls.

## Credits

Credit to Josh Swanson (Red Hat) for providing the base scripts from which this work was built on.
