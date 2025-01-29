# Operator Installation

> Commands below are executed from the root of this repository.

As with many other things, we'll need to deploy some Operators on the Hub SNO/Cluster.

1. RHACM
2. MetalLB
3. OpenShift Virtualization
4. NMState

## Installing Red Hat Advanced Cluster Management ((RH)ACM)

Most any of the IPI, HCP, ZTP, etc mechanisms work around ACM - so let's go ahead and install it.

You can do so either by clicking via the Web UI to Operators > Operator Hub, search for ACM, click Install.  Otherwise, you can also do it via the command line or via GitOps:

```bash
# Log into the Hub SNO/Cluster

# Install the ACM Operator - replace with whatever release version overlay
oc apply -k 99-operators/advanced-cluster-management/operator/overlays/release-2.12/
```

Once the Operator is installed, you can go about creating the MultiClusterHub custom resource.  This can also be done via the Web UI, just maybe set the Availability to Basic instead of High to deployer fewer replicas of things.  Of course, you could also do this via the CLI:

```bash
# Deploy the MultiClusterHub
oc apply -k 99-operators/advanced-cluster-management/instance/base/

# Optionally - deploy ACM Observability
# Assumes the use of ODF for S3 storage via Nooba
oc apply -k 99-operators/advanced-cluster-management/instance/observability/
```

Once everything is rolled out and deployed, you should be able to refresh the Web UI and have access to the ACM dashboard.  By default, ACM is not configured for a few key capabilities such as provisioning bare metal servers, but that and more will be detailed below.

---

## Installing MetalLB

> For any HCP method

Unless you're doing things in the cloud, you're going to need MetalLB to provide LoadBalancer type Services.  This is used in the Hub SNO/Cluster for all HCP types, as well as in the managed HCP bare metal clusters to provide an Ingress LoadBalancer.  The latter is automatically handled by ACM Policies and GitOps if you choose to do so.

```bash
# Install the MetalLB Operator on the Hub SNO/Cluster
oc apply -k 99-operators/metallb/operator/

# Once it's installed, create an instance
oc apply -k openshift/metallb/instance/overlays/default/
```

With that, you can go forth and configure your IPAddressPools and whatnot - more on this later.

---

## Installing Red Hat OpenShift Virtualization

> For HCP with worker VMs running on OpenShift Virtualization

In case you want to run an HCP cluster with OpenShift Virt on this Hub SNO/Cluster, go ahead and install that operator as well.  Note, this assumes you have storage deployed already either via the LVM Operator on SNO, and/or another external storage CSI installed, or ODF in an HA Hub cluster.

Install via the Web UI and Operator Hub, or via the following commands:

```bash
# Install the Operator
oc apply -k 99-operators/openshift-virtualization/overlays/default/

# Or - Install the Operator with nested virt enabled
oc apply -k 99-operators/openshift-virtualization/overlays/emulation/
```

To deploy the HyperConverged CR that initializes all the KubeVirt/OCPV things can be dependant on your environment.  The defaults are a safe starting point, but you may find it heavily customized quickly - especially in disconnected/proxied environments.

Find an example HyperConverged CR in [./manifests/ocpv/hyperconverged.yaml](./manifests/ocpv/hyperconverged.yaml) that has some considerations for private mirrors.

To install OpenShift Virtualization automatically on managed bare metal clusters, there are optional ACM Policies to do so.

## NMState

> For exposing HCP OpenShift Virt VMs to different trunked networks

Deploying NMState is optional, but handy in case you want to put VMs on your physical network and not just the Pod Network.

```bash
# Install the NMState Operator
oc apply -k openshift/nmstate/operator/

# Once it's installed create an instance of NMState
oc apply -k openshift/nmstate/instance/
```

The configuration for NMState by and large should work for you unless you have a bunch of taints all over the place.

With NMState you can do something like create a bridge to the network your machines are sitting on so you can create HCP VMs in that network.  Find a pretty straightforward way to do just that in [./ztp/components/nmstate/baremetal-network.yaml](./ztp/components/nmstate/baremetal-network.yaml)
