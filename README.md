# Simple Single-Node OpenShift Demo

Configuration-as-code for a simple SNO demo.

## Installation

Start by using the Web-based Assisted Installer.  You can access that by signing in to the [Red Hat Hybrid Cloud Console](https://console.redhat.com/) using your Red Hat credentials.

Anyone (even with a free Red Hat account) can deploy an OpenShift cluster.  The new cluster will automatically have a free 60-day trial subscription.

The Assisted Installer does a good job of walking you through the install process.  You can find [detailed instructions and prerequisites in the docs](https://docs.openshift.com/container-platform/latest/installing/installing_on_prem_assisted/installing-on-prem-assisted.html).

## Post Installation Config

The majority of the cluster configuration will be done with GitOps.  You can use "ClickOps", but that's really only suitable for experimentation.

Make sure the `StorageClass` that was created is marked as `default`.  If it isn't, run the following command:

```
oc patch storageclass <storageclass name> -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
```

To enable Multicloud Object Gateway for object storage, the SNO node needs the following label added:

```
# Find the node name.
oc get nodes

# Label the node.
oc label node <node> cluster.ocs.openshift.io/openshift-storage=''
```

### OpenShift GitOps (Argo CD)

First, install OpenShift GitOps.  This will be the tool that will configure your cluster based on config in git repos.

Log into your cluster as a cluster admin (or kubeadmin for a fresh install) and run the following command:

```
oc apply -k https://github.com/redhat-cop/gitops-catalog/openshift-gitops-operator/operator/overlays/gitops-1.13
```

Once OpenShift GitOps has finished installing, run the following command to create the "Bootstrap" Application.  This follows the "App-of-Apps" pattern, creating all of the other necessary Argo CD applications to complete the cluster configuration.

```
oc apply -k https://github.com/pittar-demos/sno-demo/gitops/argocd/clusters/sno/bootstrap
```

This will deploy and/or configure:
* [Image Registry storage](gitops/cluster-config/image-registry)
* [OpenShift GitOps](gitops/cluster-config/openshift-gitops)
* [Multicloud Object Gateway for Object Storage](gitops/cluster-config/openshift-storage)
* [Migration Toolkit for Virtualization](gitops/cluster-services/migration-toolkit-virt)
* [Cert-Manager](gitops/cluster-config/cert-manager)

### Post Install Cert-Manager Config

Normally, you would put the following config in your git repo, since domain names are not generally considered to be sensitive info (especially if you have a priate GitOps repository).  However, since this demo can be used for *any* domain, this obviously can't be hard coded up front.

The automation has handled deploying [Cert-Manager](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/index.html) as well as a defailt `Issuer` for self-signed certificates.  This `Issuer` will be used to create a root CA.  This root CA will then be used by a new cluster-wide `ClusterIssuer` to generate self-signed certificates for specific domains.

Create the following yaml file on a machine that has `oc` command line access to your cluster.  Be sure to substitute in your own `commonName`.  You can name this file whatever you like, but for this example it can be named `selfsigned-ca.yaml`.

```
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: cert-manager
spec:
  isCA: true
  commonName: apps.lab.pitt.ca
  secretName: root-secret
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: root-ca-issuer
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  ca:
    secretName: root-secret
```

Now, apply the file to create your root CA and `ClusterIssuer`.

```
oc create -f selfsigned-ca.yaml -n cert-manager
```

You can now use cert-manager to create self-signed certificates for cluster workloads.

## Notably Missing

* External-Secrets:  No secret management tool available for this demo.