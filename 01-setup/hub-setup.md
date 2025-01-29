# Hub Setup

Below are some prerequisite steps needed to be run against the Hub SNO/Cluster.

## Enable Wildcard Ingress Routes

> For HCP with OpenShift Virtualization Worker Nodes

By default, your OpenShift Ingress accepts wildcard routes on `*.apps.cluster-name.example.com` - but it can also accept routes for other domains, and even subdomain wildcards of your cluster's wildcard.

On your Hub SNO/Cluster, modify the `default` Ingress controller with the two lines below:

```yaml
---
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  # Add the following two lines
  routeAdmission:
    wildcardPolicy: WildcardsAllowed
```

## Add Internal Root CAs to Hub SNO/Cluster

If you're using an Outbound Proxy that does SSL Re-encryption (MitM'ing), or if some services like a container image registry are signed with a private Certificate Authority, you'll need to add those Root CA certificates to OpenShift - there are a variety of places we need to add our Root CAs, but a few key places for the platform are:

### Core Cluster Additional Trust Bundle

In the `openshift-config` Namespace, you should find a `user-ca-bundle` ConfigMap.  If you added additional Root Trust bundles during install you should find them here - if not, just create it at such:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: user-ca-bundle
  namespace: openshift-config
data:
  ca-bundle.crt: |
    -----BEGIN CERTIFICATE-----
    MII....ROOT CA 1
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MII....ROOT CA 2
    -----END CERTIFICATE-----
```

> This will trigger a rolling reboot of the nodes.

### Image Registries

If you're pulling from private image registries signed by a private CA, you'll need to configure that per-registry endpoint.  Do so by creating a ConfigMap as follows:


```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-ca-bundle
  namespace: openshift-config
data:
  quay-ptc.jfrog.lab.kemo.network: |
    -----BEGIN CERTIFICATE-----
    MII...Registry Root CA
    -----END CERTIFICATE-----
  docker-ptc.jfrog.lab.kemo.network: |
    -----BEGIN CERTIFICATE-----
    MII...Registry Root CA
    -----END CERTIFICATE-----
```

**Then - ** configure the Image CR to use it:

```yaml
---
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  additionalTrustedCA:
    name: image-ca-bundle
```

> **Note:** If you have configured the image registry CAs after deploying ACM/MCE/HCP then you'll need to restart the operator pods in the hypershift namespace.