# Cert-Manager

Manually managing certificates is... not fun.  It can be time consuming, error prone, and it can be difficult to remember to renew and update certificates.  Luckily, there is a fantastic automation tool for this called [cert-manager](https://cert-manager.io/).  Even better, Red Hat provides a [fully supported build of cert-manager with OpenShift Container Platoform](https://docs.openshift.com/container-platform/4.16/security/cert_manager_operator/index.html).

Normally, you would put the following config in your git repo, since domain names are not generally considered to be sensitive info (especially if you have a priate GitOps repository).  However, since this demo can be used for *any* domain, this obviously can't be hard coded up front.

The automation has handled deploying the [Cert-Manager](https://docs.openshift.com/container-platform/latest/security/cert_manager_operator/index.html) operator as well as a defailt `Issuer` for self-signed certificates.  This `Issuer` will be used to create a root CA.  This root CA will then be used by a new cluster-wide `ClusterIssuer` to generate self-signed certificates for specific domains.

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
  commonName: apps.your.cluster.domain
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

You can now use cert-manager to create self-signed certificates for cluster workloads.  You can see an example of this in the Keycloak config.