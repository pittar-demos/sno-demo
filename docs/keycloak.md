# Red Hat Build of Keycloak

[Keycloak](https://www.keycloak.org/)  is an incredibly popular open source Identity and Access Mangement system.  Red Hat is a core contributor to the Keycloak project and includes a fully supported build of Keycloak with every **OpenShift Container Platform** subscription.  The supported version is called the [Red Hat build of Keycloak](https://access.redhat.com/products/red-hat-build-of-keycloak/).

It's easy to install and manage Keycloak with the Red Hat build of Keycloak *Operator*, but you do need to include the domain name that you will use to expose Keycloak.  Normally, this is perfectly fine, since the domain of your IAM solution shouldn't change very often ;)

For a demo that anyone can deploy, that's a problem, since everyone will have their own custom domain.

For that reason, *most* of the details of Keycloak are handled by Argo CD (installing the operator, deploying Postgres), but the last pieces (creating a certificate, creating the `Keycloak` custom resource) will need to be done manually.

## Certificate

Cert-manager should already be deployed and configured if you've followed the base cluster configuration.  They `keycloak` project/namespace should already be created as well (this is where the operator has been deployed).

To create a certificate, simply create a file called `keycloak-cert.yaml` and paste in the following yaml.  Remember to change the domain to your own domain.


```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: keycloak-cert
  namespace: keycloak
spec:
  isCA: false
  commonName: "keycloak.apps.your.cluster.url" 
  secretName: keycloak-cert 
  dnsNames:
  - "keycloak.apps.your.cluster.url" 
  issuerRef:
    name: selfsigned-issuer 
    kind: ClusterIssuer
```

Now, create the certificate:

```
oc create -f keycloak-cert.yaml -n keycloak
```

Cert-manager will then create a new self-signed cert for the domain and add it to your namespace as a `Secret` named `keycloak-cert`.

## Deploy Keycloak

Now that you have a certificate (and a Postgresql database), you just need to create the Keycloak instance.

Create a yaml file named `keycloak.yaml` and paste in the following conentets.  Again, make sure you update the domain to match the domain you used when creating the certificate.

```
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: keycloak
  namespace: keycloak
spec:
  instances: 1
  db:
    vendor: postgres
    host: postgresql-db
    usernameSecret:
      name: keycloak-creds
      key: POSTGRESQL_USER
    passwordSecret:
      name: keycloak-creds
      key: POSTGRESQL_PASSWORD
  ingress:
    className: openshift-default
  http:
    tlsSecret: keycloak-cert
  hostname:
    hostname: keycloak.apps.your.cluster.url
  proxy:
    headers: xforwarded # double check your reverse proxy sets and overwrites the X-Forwarded-* headers
  resources:
    requests:
      cpu: 500m
      memory: 896Mi
    limits:
      cpu: 2
      memory: 2Gi
```

If everything is configured properly, Keycloak should be up in running in a few minutes.

You can find the randomly generated "admin" password in a `Secret` named `keycloak-initial-admin`.  Login as "admin" with the default password.  Once you are in, it's a good idea to go to the "Users" list and change the "admin" password to something that you will remember.

Keycloak is now installed and ready to go.