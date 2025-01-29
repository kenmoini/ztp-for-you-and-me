# ZTP for You and Me

This collection of assets is meant to bootstrap you in starting down the Zero Touch Provisioning path of managing OpenShift.

Note this is not the SiteConfig style ZTP - this is ZTP in the spirit of ZTP, where it's more or less just a heavy infusion of automation and GitOps into the way you do things.

Included are:

- How to get started and set things up like ACM and HCP
- HCP with Bare Metal Nodes
- HCP to OpenShift Virtualization
- Host Infrastructure to vSphere
- CA/Disconnected/Proxy Considerations

## Assumptions

There are a dozen different architectures you could use to deploy OpenShift in every which way.  For the sake of this documentation we'll assume the following:

- A Single Node OpenShift (SNO) instance, installed bare metal, to act as a Hub cluster and to run OpenShift Virtualization.  Using additional local disks via LVM Operator - or bring your own CSI.
- Two additional bare metal nodes, to be added to Advanced Cluster Management (ACM) running on the SNO Hub.  These will be used to create another two HCP clusters.
- These servers have a BMC interface with Redfish - if not, then you'll need to manually manage the boot and installation of those servers.

This makes it to where you just need 3 bare metal nodes.  You could run one HCP Bare Metal cluster with both of the other nodes, but then you have a shared storage requirement that can't be satisfied by ODF since that needs at least 3 nodes.

## Network Prerequisites

The prerequisites for OpenShift in traditional and HCP patterns are largely the same - it just kind of depends on where your DNS records go to.

| Cluster | Endpoint | VIP | DNS A Record | Notes |
|---------|----------|-----|--------------|-------|
| Hub Cluster (SNO) | App Ingress | 192.168.0.10 | *.apps.hub-cluster.example.com | SNO App VIP goes to IP of node |
| Hub Cluster (SNO) | API | 192.168.0.10 | api.hub-cluster.example.com | SNO API goes to IP of node |
| Hub Cluster (3+ Node) | App Ingress | 192.168.0.10 | *.apps.hub-cluster.example.com | Dedicated App VIP for 3+ node clusters |
| Hub Cluster (3+ Node) | API | 192.168.0.11 | api.hub-cluster.example.com | Dedicated API VIP for 3+ node clusters |

That should be pretty straightforward for most OpenShift deployments.   In HCP patterns, you still have two IPs and two DNS records to create, they just go to different places.

| Cluster | Endpoint | VIP | DNS A Record | Notes |
|---------|----------|-----|--------------|-------|
| HCP Bare Metal #1 | App Ingress | 192.168.0.21 | *.apps.hcp-bmh1-cluster.example.com | App VIP is LoadBalancer service on the managed cluster |
| HCP Bare Metal #1 | API | 192.168.0.20 | api.hcp-bmh1-cluster.example.com | API is a LoadBalancer/NodePort on the Hub cluster |
| HCP Bare Metal #2 | App Ingress | 192.168.0.26 | *.apps.hcp-bmh2-cluster.example.com | App VIP is LoadBalancer service on the managed cluster |
| HCP Bare Metal #2 | API | 192.168.0.25 | api.hcp-bmh2-cluster.example.com | API is a LoadBalancer/NodePort on the Hub cluster |
| HCP KubeVirt | App Ingress | 192.168.0.10 | *.apps.hcp-virt-cluster.apps.hub-clusterexample.com | App Ingress DNS record points to the App Ingress VIP of the Hub Cluster |
| HCP KubeVirt | API | 192.168.0.30 | api.hcp-virt-cluster.apps.hub-cluster.example.com | API is a LoadBalancer/NodePort on the Hub cluster |

## Getting Started

You'll need an OpenShift "Hub Cluster" and storage figured out for it - storage is out of scope for these documents.

You'll also either need some additional bare metal nodes to manage as a cluster, or to install the Hub cluster on a bare metal server to use with OpenShift Virtualization.

You can technically do nested virtualization, and you can technically mix a bunch of those patterns for the inception of inception combo wombo.

Anywho, to get started look in the [./01-setup](./01-setup) folder for more details on installing the various needed Operators and other needed configuration to get started.

> It's advised to fork this repo and work from there since some things will require modification

## ACM & GitOps Configuration

Before you start creating clusters you may want to create some Policies, integrate ACM and ArgoCD, etc.  This step is optional in case you're just interested in trying out Hosted Control Planes or copy/paste around a cluster for testing purposes.

Find additional details in the [./02-rhacm-config](./02-rhacm-config) folder.

## Creating a Cluster

With everything in its right place, you can now start to declaratively create clusters - so choose your own adventure:

- [./05-hcp-bmh - HCP to Bare Metal Hosts](./05-hcp-bmh)
- [./05-hcp-virt - HCP to OpenShift Virtualization](./05-hcp-virt)

---

## Disconnected Helpers

As with most things, I love to test things in disconnected/proxied environments - *because that's when we see how things break in fun, new, and exciting ways!*

A couple of pointers...

### Disconnected Helpers - Disable Insights

Just delete the credentials for `cloud.openshift.com` from the `pull-secret` Secret in the `openshift-config` namespace lol.  Unfortunately you still need it even in a disconnected environment, there are validators for its *presence*.

### Disconnected Helpers - Disable Insights Reporting

You'll probably see something like an Alert on the home page titled `PrometheusRemoteWriteBehind` with a message of `Prometheus openshift-user-workload-monitoring/prometheus-user-workload-0 remote write is 1737915340.0s behind for 56f38d:https://infogw.api.openshift.com/metrics/v1/receive.`

Also pretty easy to fix and disable telemetry - find delete the remoteWrite entry with the url `https://infogw.api.openshift.com/metrics/v1/receive` in the `user-workload-monitoring-config` ConfigMap in the `openshift-user-workload-monitoring` namespace.  By default I pretty much just delete all the data in the `config.yaml` data key.

Poof, one less alert on the dashboard!

### Disconnected Helpers - OpenShift Update Service

---

## Known Issues

### CA Cert Bundles Too Large - assisted-image-service 500 errors

You'll find an error in the `assisted-image-service` Pod in the `multicluster-engine` namespace talking about something being larger than the available space and throwing 500 errors when requesting an ISO.

This is probably because like me, you use the magic ConfigMap label that injects the whole trusted CA bundle into it - evidently that is too large.  Just use the CA chain that you need to access services.

### SuperMicro BMCs can't handle SSL lol

Evidently SuperMicro X11 servers can't use ISOs from HTTPS - https://docs.openshift.com/container-platform/4.16/edge_computing/ztp-deploying-far-edge-sites.html#ztp-troubleshooting-ztp-gitops-supermicro-tls_ztp-deploying-far-edge-sites

This patch is provided in [./01-setup/manifests/acm-mce-hcp/metal3_provisioning.yaml](./01-setup/manifests/acm-mce-hcp/metal3_provisioning.yaml)

### HyperShift Monitoring Bug

IDK - just read this: https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.11/html/clusters/cluster_mce_overview#monitor-user-workload-disconnected

That patch is provided in [./01-setup/manifests/acm-mce-hcp/hypershift-flags.yaml](./01-setup/manifests/acm-mce-hcp/hypershift-flags.yaml)