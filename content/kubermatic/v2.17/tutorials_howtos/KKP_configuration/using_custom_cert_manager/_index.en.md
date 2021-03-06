+++
title = "Using a Custom CA"
date = 2020-11-03T13:07:15+02:00
weight = 130
+++

KKP uses [cert-manager](https://cert-manager.io/) for managing all TLS certificates used for KKP
itself, as well as Dex and the Identity-Aware Proxies (IAP). The default configuration sets up
Certificate Issuers for [Let's Encrypt](https://letsencrypt.org/), but other issuers can be configured
as well. This section describes the various options for using a custom CA: using cert-manager or managing
certificates manually outside of the cluster.

## Using cert-manager

cert-manager offers a [CA Issuer](https://cert-manager.io/docs/configuration/ca/) that can automatically
create and sign certificates inside the cluster. This requires that the CA itself is stored inside the
cluster as a Kubernetes Secret, so care must be taken to prevent unauthorized access (e.g. by setting
up proper RBAC rules).

If having the CA certificate and key inside the cluster is not a problem, this approach is recommended,
as it introduces the least friction and can be achieved rather easily.

Currently the `cert-manager` Helm chart that is part of KKP does not support creating non-ACME
ClusterIssuers (i.e ClusterIssuers that do not use Let's Encrypt). New issuers must therefore be
created manually. Please follow the [description](https://cert-manager.io/docs/configuration/ca/) in
the cert-manager documentation.

Once the new ClusterIssuer has been created, KKP and the IAP need to be adjusted to use the new issuer.
For KKP, update the `KubermaticConfiguration` and configure `spec.ingress.certificateIssuer` accordingly:

```yaml
spec:
  ingress:
    certificateIssuer:
      name: my-own-ca-issuer
```

Re-apply the changed configuration and the KKP Operator will reconcile the Certificate resource,
after which cert-manager will provision a new certificate Secret.

Similarly, update your Helm `values.yaml` that is used for the Dex/IAP deployment and configure
the new issuer:

```yaml
dex:
  certIssuer:
    name: my-own-ca-issuer

iap:
  certIssuer:
    name: my-own-ca-issuer
```

Re-deploy the `iap` Helm chart to perform the changes and update the Certificate resources.

## External CA

If issuing certificates inside the cluster is not possible, static certificates can also be provided. The
cluster admin is responsible for renewing and updating them as needed. Going forward, it is assumed that
proper certificates have already been created and now need to be configured inside the cluster.

### KKP

The KKP Operator manages a single Ingress for the KKP API/dashboard. This by default includes setting up
the required annotations and spec settings for usage with cert-manager. However, if the cert-manager
integration is disabled, the cluster admin is free to manage these settings themselves.

To disable the cert-manager integration, set the `spec.ingress.certificateIssuer.name` to an empty string
in the `KubermaticConfiguration`:

```yaml
spec:
  ingress:
    certificateIssuer:
      name: ""
```

It is now possible to set `spec.tls` on the `kubermatic` Ingress to a custom certificate:

```yaml
spec:
  tls:
  - secretName: my-custom-kubermatic-cert
    hosts:
    - kubermatic.example.com
```

Refer to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls)
for details on the format for certificate Secrets.

### Dex

The same technique used for KKP is applicable to Dex as well: Set the name of the cert issuer to an empty
string to be able to configure your own certificates. Update the Helm `values.yaml` used to deploy the
chart like so:

```yaml
dex:
  certIssuer:
    name: ""
```

Re-deploy the chart and the Certificate resource will not be created anymore. You have to manually create
a `dex-tls` Secret in Dex's namespace. This Secret follows the
[same format](https://kubernetes.io/docs/concepts/services-networking/ingress/#tls) as the one for KKP's API.

### Identity-Aware Proxy

The configuration is identical to Dex: Disable the cert issuer's name and then manually create the TLS
certificates.

```yaml
iap:
  certIssuer:
    name: ""
```

For each configured Deployment (`iap.deployments`) a matching Secret needs to be created. For a Deployment
named `grafana`, the Secret needs to be called `grafana-tls`.

## Token Validation

Both the KKP API and the OAuth-Proxy from the IAP need to validate the OAuth tokens (generated by Dex,
by default). If the the token issuer uses a custom CA, this CA needs to be configured for KKP and all
IAPs.

In both cases, the full certificate chain (including intermediates) needs to be configured.

### KKP API

The token issuer (not to be confused with a cert-manager certificate issuer) is configured in the
`KubermaticConfiguration` and by default requires a valid certificate. The required adjustments for this
are the same for custom internal or external CA's.

```yaml
spec:
  auth:
    caBundle: ""
    tokenIssuer: https://example.com/dex

    # this should never be enabled in production environments
    skipTokenIssuerTLSVerify: false
```

If the certificate used for Dex is not issued by a CA that is trusted by default (e.g. Let's Encrypt),
the issuing CA's certificate chain needs to be configured in `spec.auth.caBundle`. This field is a
multiline string that should contain the PEM-encoded certificate(s), like so:

```yaml
spec:
  auth:
    caBundle: |
      -----BEGIN CERTIFICATE-----
      <certificate 1 here>
      -----END CERTIFICATE-----

      -----BEGIN CERTIFICATE-----
      <certificate 2 here>
      -----END CERTIFICATE-----

      -----BEGIN CERTIFICATE-----
      <certificate 3 here>
      -----END CERTIFICATE-----
```

Do note that if the [OIDC Endpoint Feature]({{< ref "../../OIDC_Provider_Configuration" >}}) is enabled in KKP, this CA bundle
is also configured for the Kubernetes apiserver, so that it can also validate the tokens issued by Dex.

### IAP

The certificate chain can be put into a Kubernetes Secret and then be referred to from the `values.yaml`.
Create a Secret inside the IAP's namespace (`iap` by default) and then update your `values.yaml` like so:

```yaml
iap:
  customProviderCA:
    secretName: my-ca-secret
    secretKey: ca.crt
```

Re-deploy the `iap` Helm chart to apply the changes.

## Wildcard Certificates

Generally the KKP stack is built to use dedicated certificates for each Ingress / application, but it's
possible to instead configure a single (usually wildcard) certificate in nginx that will be used as the
default certificate for all domains.

As with all other custom certificates, create a new Secret with the certificate and private key in it,
and then adjust your Helm `values.yaml` to configure nginx like so:

```yaml
nginx:
  extraArgs:
    # The value of this flag is in the form "namespace/name".
    - '--default-ssl-certificate=mynamespace/mysecret'
```

Redeploy the `nginx-ingress-controller` Helm chart to enable the changes.
