---
date: 2021-02-12T18:00:00
description: "Perform a disconnected installation of the Nvidia GPU Operator on Openshift"
#featured_image: "/images/keycloak_logo.png"
tags: ["Openshift","Nvidia"]
# categories: "POC"
title: "Installing the Nvidia GPU Operator on Openshift 4"
comment : false
---

A disconnected installation of the gpu-operator on Openshift is not officially documented. If you attempt to install the Operator disconnected the pods fail to run as they are unable to pull the required rpms to build the driver for the specific kernel version of the CoreOS node on which they are running on. The operator can therefore be made to work by hosting the required rpm packages in a local mirror.

This article will not concentrate too much on getting the operator and its images available to the cluster, but more on solving the issue of the installation when the operator and images are available.

Nvidia does have some documentation on a disconnected 'air gapped' installation for this operator. This can be found here: [Considerations to Install in Air-Gapped Clusters](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html).

## Container Images

The below steps expect that the existence of a local mirror registry and a custom operator catalog image and that these already setup and working correctly. The documentation for this can be found here: [Using Operator Lifecycle Manager on restricted networks](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-restricted-networks.html).

Mirror the required images in your local image registry. You should also be able to see the images listed here: [gpu-operator.clusterserviceversion.yaml](https://github.com/NVIDIA/gpu-operator/blob/master/bundle/manifests/gpu-operator.clusterserviceversion.yaml#L128). At the time of writing the latest version was 1.5.2 and this looks like this:

```yaml
relatedImages:
  - name: gpu-operator-image
    image: nvcr.io/nvidia/gpu-operator@sha256:679fea62eb2c207d26354ac088fbe4625457a329dee080d90479a411603eb694
  - name: dcgm-exporter-image
    image: nvcr.io/nvidia/k8s/dcgm-exporter@sha256:85016e39f73749ef9769a083ceb849cae80c31c5a7f22485b3ba4aa590ec7b88
  - name: container-toolkit-image
    image: nvcr.io/nvidia/k8s/container-toolkit@sha256:81295a9eca36cbe5d94b80732210b8dc7276c6ef08d5a60d12e50479b9e542cd
  - name: driver-image
    image: nvcr.io/nvidia/driver@sha256:324e9dc265dec320207206aa94226b0c8735fd93ce19b36a415478c95826d934
  - name: device-plugin-image
    image: nvcr.io/nvidia/k8s-device-plugin@sha256:f7bf5955a689fee4c1c74dc7928220862627adc97e00a4b585f9c31965e79625
  - name: gpu-feature-discovery-image
    image: nvcr.io/nvidia/gpu-feature-discovery@sha256:8d068b7b2e3c0b00061bbff07f4207bd49be7d5bfbff51fdf247bc91e3f27a14
  - name: cuda-sample-image
    image: nvcr.io/nvidia/k8s/cuda-sample@sha256:2a30fe7e23067bc2c3f8f62a6867702a016af2b80b9f6ce861f3fea4dfd85bc2
  - name: dcgm-init-container-image
    image: nvcr.io/nvidia/cuda@sha256:ed723a1339cddd75eb9f2be2f3476edf497a1b189c10c9bf9eb8da4a16a51a59
```

**WARNING**: If getting the images from the above CSV yaml be sure to use the correct version for the operator you are planning on using.

Then follow [image configuration](https://docs.openshift.com/container-platform/4.6/openshift_images/image-configuration.html) to configure the registrySources of OpenShift to pull those images from the mirror registry.

The disconnected documentation from Nvidia notes Ubuntu based images therefore make sure to use the correct images for your use case e.g. for Openshift use the RHEL based images. [nvidia documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/getting-started.html).


## Packages

The following packages need to exist in a local mirror.  Documentation for setting up a custom local mirror for these packages can be found here: [Creating Local Mirrors for Updates or Installs](https://wiki.centos.org/HowTos/CreateLocalMirror).

The matching of rpm to the kernel version of the RHCOS nodes is quite important. Therefore, to ensure this make sure to find the $HOST_ARCH and $GPU_NODE_KERNEL_VERSION by running `oc describe node` for a bare metal node that has a GPU.

```bash
elfutils-libelf.${HOST_ARCH}
elfutils-libelf-devel.${HOST_ARCH}
kernel-headers-${GPU_NODE_KERNEL_VERSION}
kernel-devel-${GPU_NODE_KERNEL_VERSION}
kernel-core-${GPU_NODE_KERNEL_VERSION}
```

These above packages are needed to run the driver container, as can be seen [here](https://gitlab.com/nvidia/container-images/driver/-/blob/master/rhel8/nvidia-driver).


# Installation

With the mirrors for both the container images and rpm packages in place, the installation can commence.

Firstly, install the NodeFeatureDiscovery and NVIDIA GPU Operators from the operator hub. Documentation for this can be found here: [Installing via OpenShift OperatorHub](https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html#openshift-gpu-support-install-via-operatorhub).

However, do not create the cluster policy for the NVIDIA GPU Operator yet.

The next step is to add the repository configuration for the driver container. Create a ConfigMap named 'repo-config' in the 'gpu-operator-resources' namespace containing the repository configuration file.

```bash
oc create configmap repo-config -n gpu-operator-resources
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: repo-config
  namespace: gpu-operator-resources
data:
  local8.repo: |-
    [nvidia-operator]
    name=nvidia-operator
    baseurl=http://host/repo/nvidia-operator/
    enabled=1
    gpgcheck=0
    gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Now create the cluster policy for the NVIDIA GPU Operator (following the earlier documentation [here](https://docs.nvidia.com/datacenter/kubernetes/openshift-on-gpu-install-guide/index.html#openshift-gpu-support-install-via-operatorhub)) with the default names but be sure to update the driver section to map to the repo-config config map. This section is documented below.

The driver version comes from the version of RHCOS you are using. At the time of writing the driver version, 450.80.02 was the latest and mapped against RHCOS 4.6. [nvidia/driver:450.80.02-rhcos4.6](https://hub.docker.com/layers/nvidia/driver/450.80.02-rhcos4.6/images/sha256-324e9dc265dec320207206aa94226b0c8735fd93ce19b36a415478c95826d934?context=explore). Further driver images can be found here [Nvidia driver images](https://gitlab.com/nvidia/container-images/driver).

```yaml
driver:
  repository: 'repo.local:9443/nvidia'
  image: driver
  version: 450.80.02
  repoConfig:
    configMapName: repo-config
    destinationDir: /etc/yum.repos.d
```

The final cluster policy might look something like the below:

```yaml
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  dcgmExporter:
    nodeSelector: {}
    imagePullSecrets: []
    resources: {}
    affinity: {}
    podSecurityContext: {}
    repository: 'repo.local:9443/nvidia'
    securityContext: {}
    version: 2.0.13-2.1.0-ubi8
    image: dcgm-exporter
    tolerations: []
  devicePlugin:
    nodeSelector: {}
    imagePullSecrets: []
    resources: {}
    affinity: {}
    podSecurityContext: {}
    repository: 'repo.local:9443/nvidia'
    securityContext: {}
    version: v0.7.0-ubi8
    image: k8s-device-plugin
    tolerations: []
  driver:
    nodeSelector: {}
    imagePullSecrets: []
    resources: {}
    affinity: {}
    podSecurityContext: {}
    repository: 'repo.local:9443/nvidia'
    securityContext: {}
    repoConfig:
      configMapName: repo-config
      destinationDir: /etc/yum.repos.d
    version: 450.80.02
    image: driver
    tolerations: []
  gfd:
    nodeSelector: {}
    imagePullSecrets: []
    resources: {}
    affinity: {}
    podSecurityContext: {}
    repository: 'repo.local:9443/nvidia'
    securityContext: {}
    version: v0.2.1
    image: gpu-feature-discovery
    sleepInterval: 60s
    tolerations: []
    migStrategy: none
  operator:
    defaultRuntime: crio
    deployGFD: true
    validator:
      image: cuda-sample
      imagePullSecrets: []
      repository: 'repo.local:9443/nvidia'
      version: vectoradd-cuda10.2-ubi8
  toolkit:
    nodeSelector: {}
    imagePullSecrets: []
    resources: {}
    affinity: {}
    podSecurityContext: {}
    repository: 'repo.local:9443/nvidia'
    securityContext: {}
    version: 1.3.0-ubi8
    image: container-toolkit
    tolerations: []
```

Once the cluster policy for the NVIDIA GPU Operator has been successfully created verify that the pods are building the driver and are starting up okay. At this point, the operator should now be working in a disconnected environment.

## Upgrades

The only thing to note is to be careful when performing cluster upgrades. If the kernel of the CoreOS node changes when performing an upgrade make sure to have the required kernel-headers, kernel-devel and kernel-core packages for the new kernel version available.
