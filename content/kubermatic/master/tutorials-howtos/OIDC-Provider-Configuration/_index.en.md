+++
title = "OIDC Provider Configuration"
date = 2018-06-21T14:07:15+02:00
weight = 14

+++

This manual explains how to configure a custom OIDC provider to use with Kubermatic Kubernetes Platform (KKP).

### Default Configuration

When nothing is configured, KKP uses `https://<domain>/dex` as the OIDC provider
URL, which by default points to Dex. The domain is taken from the
[KubermaticConfiguration]({{< ref "../../tutorials-howtos/kkp-configuration" >}}).

When redirecting users to the OIDC provider for login into the KKP dashboard, KKP
adds the following parameters to the base URL:

- `&response_type` is set to `id_token`
- `&client_id` is set to `kubermatic`
- `&redirect_uri` is set to `https://<domain>/projects` which is root view of the KKP dashboard
- `&scope` is set to `openid email profile groups`
- `&nonce` is randomly generated, 32 character string to prevent replay attacks

### Custom Configuration

The default configuration can be changed as KKP supports other OIDC providers as well. This
involves updating the KKP dashboard and API using the `KubermaticConfiguration` CRD on the
master cluster. The used configuration can be retrieved using `kubectl`:

```bash
kubectl -n kubermatic get kubermaticconfigurations
#NAME         AGE
#kubermatic   2h

kubectl -n kubermatic get kubermaticconfiguration kubermatic -o yaml
#apiVersion: kubermatic.k8c.io/v1
#kind: KubermaticConfiguration
#metadata:
#  finalizers:
#  - operator.kubermatic.io/cleanup
#  name: kubermatic
#  namespace: kubermatic
#spec:
#  auth:
#    issuerClientSecret: abcd1234
#    issuerCookieKey: wxyz9876
#    serviceAccountKey: 2468mnop
#  ...
```

There are two sections to update.

#### API Configuration

The KKP API validates the given token for authentication and therefore needs to be able to
find the new token issuer. The relevant fields are under `spec.auth` and the following snippet
demonstrates the default values:

```yaml
spec:
  auth:
    clientID: kubermatic
    issuerClientID: kubermaticIssuer
    issuerClientSecret: ""
    issuerCookieKey: ""
    issuerRedirectURL: https://<domain>/api/v1/kubeconfig
    serviceAccountKey: ""
    skipTokenIssuerTLSVerify: false
    tokenIssuer: https://<domain>/dex
```

The `tokenIssuer` needs to be updated, the rest can be left out if the default values are
used. This gives us:

```yaml
spec:
  auth:
    tokenIssuer: 'https://keycloak.kubermatic.test/auth/realms/test'
```

#### UI Configuration

The KKP dashboard needs to know where to redirect the user to in order to perform a
login. This can be set by setting a `spec.ui.config` field, containing JSON. This is where
various UI-related options can be set, among them:

- `oidc_provider_url` to change the base URL of the OIDC provider
- `oidc_provider_scope` to change the scope of the OIDC provider (the `scope` URL parameter)

A configuration of a custom OIDC provider may look like this:

```yaml
spec:
  ui:
    config: |
      {
        "oidc_provider_url": "https://keycloak.kubermatic.test/auth/realms/test/protocol/openid-connect/auth",
        "oidc_provider_scope": "openid email profile roles"
      }
```

### Applying the Changes

Edit the KubermaticConfiguration either directly via `kubectl edit` or apply it from a YAML
file by using `kubectl apply`. The KKP Operator will pick up on the changes and
reconfigure the components accordingly. After a few seconds the new pods should be up and
running.

{{% notice note %}}
If you are using _Keycloak_ as a custom OIDC provider, make sure that you set the option `Implicit Flow Enabled: On`
on the `kubermatic` and `kubermaticIssuer` clients. Without this option, you won't be properly
redirected to the login page.
{{% /notice %}}
