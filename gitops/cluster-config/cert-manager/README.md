# Cert-Manager Operator

This expects a secret with your route53 credentials.  For example:

```
access-key-id=<aws access key id>
secret-access-key=<aws secret access key>
```

Don't check this sort of secret into Git!  I use the External Secrets operator to manage my secrets, but there are many other options.