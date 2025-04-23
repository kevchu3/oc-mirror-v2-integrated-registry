# Use oc-mirror v2 for an OpenShift Integrated Registry



![GitHub](https://img.shields.io/github/license/kevchu3/oc-mirror-v2-integrated-registry?color=blue&style=plastic)

## Background

The following procedures are designed for an airgapped OpenShift cluster to mirror images to the OpenShift integrated registry, to reduce setup time and eliminate network connectivity to an external registry.  Mirroring uses the oc-mirror OpenShift CLI (oc) plugin, which mirrors all required OpenShift Container Platform content and other images to your mirror registry.  We'll use the oc-mirror v2 plugin ([refer to the docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/disconnected_environments/mirroring-in-disconnected-environments#about-installing-oc-mirror-v2) to mirror to the cluster's OpenShift integrated registry, typically available at `image-registry.openshift-image-registry.svc.cluster.local:5000`.

## Instructions

Create new projects `oc-mirror` for the mirror job and `oc-mirror-images` for the mirrored imagestreams.  Apply the manifests in this repository:
```
oc new-project oc-mirror         # mirror job
oc new-project oc-mirror-images  # mirrored imagestreams
oc apply -f manifests
```

## Credits

Credit to Josh Swanson (Red Hat) for providing the base scripts from which this work was built on.
