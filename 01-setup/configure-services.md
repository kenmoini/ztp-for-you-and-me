# Configure Services

By default, RHACM doesn't have everything turned on - we need to activate things such as the Assisted Service and Metal3 Provisioner.

## Disconnected/Proxied HTTP Mirror for RHCOS Images

> For disconnected/proxied environments

By default, RHACM doesn't enable the Assisted Service that manages RHCOS images.  To do so is pretty easy, unless you're in a disconnected/proxied environment cause there's no way to define an outbound proxy to be used by it - and if you're fully disconnected then you need to mirror the RHCOS image parts any way.

To make this easy in an OpenShift hub cluster that has outbound proxy access, there is a `http-mirror` application that can take in proxy configuration, download assets, and host them to be then used by the Assisted Service.  You can find this in the [./manifests/http-mirror/](./manifests/http-mirror/) directory with the defaults already set up for mirroring 4.17, 4.16, and 4.15 RHCOS image assets.

## Create Mirror Registry ConfigMap

If you're using some mirror image registries and/or custom Root CAs that sign it, you'll need to create a ConfigMap before deploying the Assisted Service - modify as needed to point to your mirror endpoint paths:

```yaml
# 01-setup/manifests/acm-mce-hcp/mce_mirror-registry-config.yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: mirror-registry-config
  namespace: multicluster-engine
data:
  # Root CA for your Image Registry
  ca-bundle.crt: |
    -----BEGIN CERTIFICATE-----
    MII...
    -----END CERTIFICATE-----
  registries.conf: |
    unqualified-search-registries = ["quay-ptc.jfrog.lab.kemo.network", "registry.access.redhat.com", "quay.io", "docker.io"]
    [[registry]]
      prefix = ""
      location = "quay.io/openshift-release-dev/ocp-release"
      mirror-by-digest-only = true
      [[registry.mirror]]
        location = "quay-ptc.jfrog.lab.kemo.network/openshift-release-dev/ocp-release"
    [[registry]]
      prefix = ""
      location = "quay.io/ocpmetal/assisted-installer"
      mirror-by-digest-only = false
      [[registry.mirror]]
        location = "quay-ptc.jfrog.lab.kemo.network/ocpmetal/assisted-installer"
    [[registry]]
      prefix = ""
      location = "quay.io/ocpmetal/assisted-installer-agent"
      mirror-by-digest-only = false
      [[registry.mirror]]
        location = "quay-ptc.jfrog.lab.kemo.network/ocpmetal/assisted-installer-agent"
    [[registry]]
      prefix = ""
      location = "quay.io/openshift-release-dev/ocp-v4.0-art-dev"
      mirror-by-digest-only = true
      [[registry.mirror]]
        location = "quay-ptc.jfrog.lab.kemo.network/openshift-release-dev/ocp-v4.0-art-dev"
    [[registry]]
      prefix = ""
      location = "quay.io"
      mirror-by-digest-only = false
      [[registry.mirror]]
        location = "quay-ptc.jfrog.lab.kemo.network"
    [[registry]]
      prefix = ""
      location = "registry.redhat.io"
      mirror-by-digest-only = false
      [[registry.mirror]]
        location = "registry-redhat-ptc.jfrog.lab.kemo.network"
    [[registry]]
      prefix = ""
      location = "registry.connect.redhat.com"
      mirror-by-digest-only = false
      [[registry.mirror]]
        location = "registry-connect-redhat-ptc.jfrog.lab.kemo.network"
```

## Create Hypershift Monitoring Patch

There's a monitoring bug with Hypershift and how it reports things - there's a ConfigMap patch that can fix this:

```yaml
# 01-setup/manifests/acm-mce-hcp/hypershift-flags.yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: hypershift-operator-install-flags
  namespace: local-cluster
data:
  installFlagsToAdd: ""
  installFlagsToRemove: "--enable-uwm-telemetry-remote-write"
```

More Info: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html/clusters/cluster_mce_overview#monitor-user-workload-disconnected

## Define ClusterImageSets

In case you're in a disconnected environment, or you just want to manage the versions of OpenShift you deploy, you'll need to create some ClusterImageSets:

```yaml
# 01-setup/manifests/acm-mce-hcp/mce_clusterimagesets.yaml
---
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v41712
  labels:
    channel: stable
    visible: 'true'
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.17.12-x86_64
---
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v41543
  labels:
    channel: stable
    visible: 'true'
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.15.43-x86_64
---
apiVersion: hive.openshift.io/v1
kind: ClusterImageSet
metadata:
  name: openshift-v41630
  labels:
    channel: stable
    visible: 'true'
spec:
  releaseImage: quay.io/openshift-release-dev/ocp-release:4.16.30-x86_64
```

## Define the Metal3 Provisioning Service

> For using bare metal nodes

Deploy the Metal3 Provisioning service that handles Redfish orchestration and asset inspection:

```yaml
# 01-setup/manifests/acm-mce-hcp/metal3_provisioning.yaml
---
apiVersion: metal3.io/v1alpha1
kind: Provisioning
metadata:
  name: provisioning-configuration
spec:
  provisioningNetwork: "Disabled"
  watchAllNamespaces: true
  # https://docs.openshift.com/container-platform/4.16/edge_computing/ztp-deploying-far-edge-sites.html#ztp-troubleshooting-ztp-gitops-supermicro-tls_ztp-deploying-far-edge-sites
  disableVirtualMediaTLS: true
```

## Deploy the Assisted Service

The Assisted Service handles creating installation ISOs for our managed clusters - with everything in place, we can now deploy it:

```yaml
# 01-setup/manifests/acm-mce-hcp/mce_agentserviceconfig.yaml
---
apiVersion: agent-install.openshift.io/v1beta1
kind: AgentServiceConfig
metadata:
  name: agent
  annotations:
    unsupported.agent-install.openshift.io/assisted-service-configmap: "assisted-service-config"
    unsupported.agent-install.openshift.io/assisted-image-service-skip-verify-tls: "true"
###
spec:
  databaseStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  filesystemStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi
  imageStorage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 50Gi

# ========================================================================================
# Disconnected/Private Registry Configuration
# ========================================================================================
#  mirrorRegistryRef:
#    name: mirror-registry-config
#  osImages:
#    # The following examples point to the HTTP mirror mentioned at the top of this document
#    - openshiftVersion: "4.17"
#      version: "417.94.202501071621-0"
#      url: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.17/latest/rhcos-live.x86_64.iso"
#      rootFSUrl: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.17/latest/rhcos-live-rootfs.x86_64.img"
#      cpuArchitecture: x86_64
#
#    - openshiftVersion: "4.16"
#      version: "416.94.202501030250-0"
#      url: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.16/latest/rhcos-live.x86_64.iso"
#      rootFSUrl: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.16/latest/rhcos-live-rootfs.x86_64.img"
#      cpuArchitecture: x86_64
#
#    - openshiftVersion: "4.15"
#      version: "415.92.202501080724-0"
#      url: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.15/latest/rhcos-live.x86_64.iso"
#      rootFSUrl: "http://ztp-mirror.multicluster-engine.svc.cluster.local:8080/pub/openshift-v4/x86_64/dependencies/rhcos/4.15/latest/rhcos-live-rootfs.x86_64.img"
#      cpuArchitecture: x86_64
```
