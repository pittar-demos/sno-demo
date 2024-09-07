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

### Post Install Configuration

Optional configuration that requires a small amount of manual work (in order to avoid hard-coding domain names into these files):

1. [cert-manager](docs/cert-manager.md):  Configure a self-signed `ClusterIssuer` to create certificates for workloads.
2. [Keycloak](docs/keycloak.md):  An Identity and Access Management platform (requires cert-manager to be configured first).


## Notably Missing

* External-Secrets:  No secret management tool available for this demo.