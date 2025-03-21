---
title: Mutual TLS
---

{% if_version gte:2.9.x %}
{% tip %}
If you want to configure version, ciphers or per service permissive / strict mode check out [`MeshTLS`](/docs/{{ page.release }}/policies/meshtls)
{% endtip %}
{% endif_version %}


This policy enables automatic encrypted mTLS traffic for all the services in a [`Mesh`](/docs/{{ page.release }}/production/mesh/), as well as assigning an identity to every data plane proxy. {{site.mesh_product_name}} supports different types of CA backends as well as automatic certificate rotation.

{{site.mesh_product_name}} ships with the following CA (Certificate Authority) supported backends:

- [builtin](#usage-of-builtin-ca): it automatically auto-generates a CA root certificate and key, that are also being automatically stored as a [Secret](/docs/{{ page.release }}/production/secure-deployment/secrets/).
- [provided](#usage-of-provided-ca): the CA root certificate and key are being provided by the user in the form of a [Secret](/docs/{{ page.release }}/production/secure-deployment/secrets/).

Once a CA backend has been specified, {{site.mesh_product_name}} will then automatically generate a certificate for every data plane proxy in the [`Mesh`](/docs/{{ page.release }}/production/mesh/). The certificates that {{site.mesh_product_name}} generates are SPIFFE compatible and are used for AuthN/Z use-cases in order to identify every workload in our system.

{% tip %}
The certificates that {{site.mesh_product_name}} generates have a SAN set to `spiffe://<mesh name>/<service name>`. When {{site.mesh_product_name}} enforces policies that require an identity like {% if_version lte:2.5.x %}[`TrafficPermission`](/docs/{{ page.release }}/policies/traffic-permissions){% endif_version %}{% if_version gte:2.6.x %}[`MeshTrafficPermission`](/docs/{{ page.release }}/policies/meshtrafficpermission){% endif_version %} it will extract the SAN from the client certificate and use it to match the service identity.
{% endtip %}

Remember that by default mTLS **is not** enabled and needs to be explicitly enabled as described below. Also remember that by default when mTLS is enabled all traffic is denied **unless** a {% if_version lte:2.5.x %}[`TrafficPermission`](/docs/{{ page.release }}/policies/traffic-permissions){% endif_version %}{% if_version gte:2.6.x %}[`MeshTrafficPermission`](/docs/{{ page.release }}/policies/meshtrafficpermission){% endif_version %} policy is being configured to explicitly allow traffic across proxies.

{% tip %}
Always make sure that a {% if_version lte:2.5.x %}[`TrafficPermission`](/docs/{{ page.release }}/policies/traffic-permissions){% endif_version %}{% if_version gte:2.6.x %}[`MeshTrafficPermission`](/docs/{{ page.release }}/policies/meshtrafficpermission){% endif_version %} resource is present before enabling mTLS in a Mesh in order to avoid unexpected traffic interruptions caused by a lack of authorization between proxies.
{% endtip %}

To enable mTLS we need to configure the `mtls` property in a [`Mesh`](/docs/{{ page.release }}/production/mesh/) resource. We can have as many `backends` as we want, but only one at a time can be enabled via the `enabledBackend` property.

If `enabledBackend` is missing or empty, then mTLS will be disabled for the entire Mesh.

## Usage of "builtin" CA

This is the fastest and simplest way to enable mTLS in {{site.mesh_product_name}}.

With a `builtin` CA backend type, {{site.mesh_product_name}} will dynamically generate its own CA root certificate and key that it uses to automatically provision (and rotate) certificates for every replica of every service.

We can specify more than one `builtin` backend with different names, and each one of them will be automatically provisioned with a unique pair of certificate + key (they are not shared).

To enable a `builtin` mTLS for the entire Mesh we can apply the following configuration:

{% if_version gte:2.6.x %}
{% warning %}
Starting with {{site.mesh_product_name}} version 2.6.0, we no longer create a default [`TrafficPermission`](/docs/{{ page.release }}/policies/traffic-permissions) policy that's essential for traffic to function after enabling mTLS. To prevent disruption of your traffic, it's highly recommended to add a specific or default [`MeshTrafficPermission`](/docs/{{ page.release }}/policies/meshtrafficpermission#allow-all) policy before enabling mTLS. This policy will allow communication between your applications.
{% endwarning %}
{% endif_version %}

{% tabs %}
{% tab Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
      - name: ca-1
        type: builtin
        dpCert:
          rotation:
            expiration: 1d
        conf:
          caCert:
            RSAbits: 2048
            expiration: 10y
```

We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}

{% tab Universal %}

```yaml
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: builtin
      dpCert:
        rotation:
          expiration: 1d
      conf:
        caCert:
          RSAbits: 2048
          expiration: 10y
```

We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.release }}/reference/http-api).
{% endtab %}
{% endtabs %}

A few considerations:

- The `dpCert` configuration determines how often {{site.mesh_product_name}} should automatically rotate the certificates assigned to every data plane proxy.
- The `caCert` configuration determines a few properties that {{site.mesh_product_name}} will use when auto-generating the CA root certificate.

### Storage of Secrets

When using a `builtin` backend {{site.mesh_product_name}} automatically generates a root CA certificate and key that are being stored as a {{site.mesh_product_name}} [Secret resource](/docs/{{ page.release }}/production/secure-deployment/secrets/) with the following name:

- `{mesh name}.ca-builtin-cert-{backend name}` for the certificate
- `{mesh name}.ca-builtin-key-{backend name}` for the key

On Kubernetes, {{site.mesh_product_name}} secrets are being stored in the `{{site.mesh_namespace}}` namespace, while on Universal they are being stored in the underlying [store](/docs/{{ page.release }}/documentation/configuration#store) configured in `kuma-cp`.

We can retrieve the secrets via `kumactl` on both Universal and Kubernetes, or via `kubectl` on Kubernetes only:

{% tabs %}
{% tab kumactl %}

The following command can be executed on any {{site.mesh_product_name}} backend:

```sh
kumactl get secrets [-m MESH]
# MESH      NAME                           AGE
# default   default.ca-builtin-cert-ca-1   1m
# default   default.ca-builtin-key-ca-1    1m
```

{% endtab %}
{% tab kubectl %}

The following command can be executed only on Kubernetes:

```sh
kubectl get secrets \
    -n {{site.mesh_namespace}} \
    --field-selector='type=system.kuma.io/secret'
# NAME                             TYPE                                  DATA   AGE
# default.ca-builtin-cert-ca-1     system.kuma.io/secret                 1      1m
# default.ca-builtin-key-ca-1      system.kuma.io/secret                 1      1m
```

{% endtab %}
{% endtabs %}

## Usage of "provided" CA

If you choose to provide your own CA root certificate and key, you can use the `provided` backend. With this option, you must also manage the certificate lifecycle yourself.

Unlike the `builtin` backend, with `provided` you first upload the certificate and key as [Secret resource](/docs/{{ page.release }}/production/secure-deployment/secrets/), and then reference the Secrets in the mTLS configuration.

{{site.mesh_product_name}} then provisions data plane proxy certificates for every replica of every service from the CA root certificate and key.

Sample configuration:

{% tabs %}
{% tab Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
      - name: ca-1
        type: provided
        dpCert:
          rotation:
            expiration: 1d
        conf:
          cert:
            secret: name-of-secret
          key:
            secret: name-of-secret
```

We will apply the configuration with `kubectl apply -f [..]`.
{% endtab %}

{% tab Universal %}

```yaml
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: provided
      dpCert:
        rotation:
          expiration: 1d
      conf:
        cert:
          secret: name-of-secret
        key:
          secret: name-of-secret
```

We will apply the configuration with `kumactl apply -f [..]` or via the [HTTP API](/docs/{{ page.release }}/reference/http-api).
{% endtab %}
{% endtabs %}

A few considerations:

- The `dpCert` configuration determines how often {{site.mesh_product_name}} should automatically rotate the certificates assigned to every data plane proxy.
- The Secrets must exist before referencing them in a `provided` backend.

### Intermediate CA

You can also work with an Intermediate CA with a `provided` backend. Generate the certificate and place it first in certificate file, before the certificate from the root CA. The certificate from the root CA should start on a new line. Then create the secret to specify in the `cert` section of the config. The secret for the `key` should contain only the private key of the certificate from the intermediate CA.

You can chain certificates from multiple intermediate CAs the same way. Place the certificate from the closest CA at the top of the cert file, followed by certificates in order up the certificate chain, then generate the secret to hold the contents of the file.

Sample certificate file for a single intermediate CA:

```
-----BEGIN CERTIFICATE-----
MIIDdjCCAl6gAwIBAgICEAEwDQYJKoZIhvcNAQELBQAwRDELMAkGA1UEBhMCR0Ix
EDAOBgNVBAgMB0VuZ2xhbmQxEjAQBgNVBAoMCUFsaWNlIEx0ZDEPMA0GA1UEAwwG
S3VtYUNBMB4XDTIxMDUxMjEzMzU1MVoXDTMxMDUxMDEzMzU1MVowUDELMAkGA1UE
BhMCR0IxEDAOBgNVBAgMB0VuZ2xhbmQxEjAQBgNVBAoMCUFsaWNlIEx0ZDEbMBkG
A1UEAwwSS3VtYUludGVybWVkaWF0ZUNBMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8A
MIIBCgKCAQEA1VzY9vOr8+SINzqA8Rwk4bpeex32Zn9BGAUTweRgomQC7Yfzrm6/
Vk74/T/46n3FydpdEZTdoFKCF8EsA0eqAEfWi6tu7D41GOUFUYpdRJBJEq+HE17Q
N8SFMquy8NhCtK8th8ytSu2ThvCOq1MHT5WjtQUmRGSJMlcfWA5TsCIK0Sb3cSf3
jadjEqcmcvJN6Xa0Y0VivcPg5eB+We7BNnp4ogqmZw0veoPjc14HVZpqxrra9Yez
DRai6rnHqDjnkMMhe9MmSkCKD9Ldwduq0ZfuOQFIBOaX+4MKUyDN4tTMCcRRl/Nl
A4JgrNNWCFfUQV0VmQ0Tc8+cn/+gokHAZwIDAQABo2YwZDAdBgNVHQ4EFgQUGNjz
Te727HX4AqZDMn1L9XzkTaYwHwYDVR0jBBgwFoAUSu2E4Ue5aPzdWQCCNp36Pf3i
YbcwEgYDVR0TAQH/BAgwBgEB/wIBADAOBgNVHQ8BAf8EBAMCAQYwDQYJKoZIhvcN
AQELBQADggEBACuOczJlf4wcT9rfAIrZHuI5aCzYTKOxJllhN5e/eEhMYpsox6Zb
4CZXS3wdJ3fVugddLWDzIAjrNE1DrOpugUPurNIpHsT6u+SHFXkRsXyHFfMA+CZJ
0tOYEtP1r3BnqsY/nh0GJqHJxaJolEaqFaKgKTQPTinOxTKFxsHa1OHlsvkdxvot
d2BQhPQYWes3LMPxtGhS5kwKaXaB3gzTnzjGvgGNeJ+l0AiWqXkivixpox3/6mMa
90mwssl4sRQQLR1kLFU4hwghNm52Pk7o7HSTEXsnB+ZhHB9skpetY6R4uKWh8xap
Xmj4PDrAA5OKZzSO7Yhdt0vXPOIrjShMxvA=
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDbjCCAlagAwIBAgIJALDMMa9rXKLPMA0GCSqGSIb3DQEBCwUAMEQxCzAJBgNV
BAYTAkdCMRAwDgYDVQQIDAdFbmdsYW5kMRIwEAYDVQQKDAlBbGljZSBMdGQxDzAN
BgNVBAMMBkt1bWFDQTAeFw0yMTA1MTIxMzE2MjFaFw00MTA1MDcxMzE2MjFaMEQx
CzAJBgNVBAYTAkdCMRAwDgYDVQQIDAdFbmdsYW5kMRIwEAYDVQQKDAlBbGljZSBM
dGQxDzANBgNVBAMMBkt1bWFDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoC
ggEBANCJqVJjYOWFUZcdhrfBxgoCZNE+LFq9sieP2yRGrYzJsCdwphH6L7GsWds8
VjlobfIP4nA23TJiMWlsx126r7pSRbVEq8/JoNa0vMspEmtjHZhSweIXWXX7o8V+
FRKbCW5NyqGiHF0ScE4VpNc3uWCA2zcaU80G9SAKI83cUjnp2JzLPMqppQ+pj6Hs
G+8322FPA2L11fsCAqdCW+gwJWpKzlfBPyeNTUOMpcP8n+Yjcah4tqcCY2PZ7nH7
cZN1vHGhT5/Pn3VRaNHUq4y1Zn/wJnjlOcD4DbVFXYpYIlPx+yAs56FXd3a7Imfg
56HzOLOZcDY/+Sxy7J2Pq8cipTcCAwEAAaNjMGEwHQYDVR0OBBYEFErthOFHuWj8
3VkAgjad+j394mG3MB8GA1UdIwQYMBaAFErthOFHuWj83VkAgjad+j394mG3MA8G
A1UdEwEB/wQFMAMBAf8wDgYDVR0PAQH/BAQDAgEGMA0GCSqGSIb3DQEBCwUAA4IB
AQCBqj9F+OJZXifyUGq9bAiybpP9RYnKd0JCiByvO/S95v6Bz9RnwrvgN75mzpPd
OM51MYKyBLFKJpvrmyQ+njcsVMnv//MH7cHE8h6WkwP9IggNg0K21J1zkS8ApfTw
7buUemZn6NFqHgysAUnWq8WM8YxfEErubbTCm6wslTLzLdblBGLjh7qOzDGh8n0e
BjqWgCYjbEsB4tDxjfSjLjSyldvnIMTyWrA8a/1iCNDXj0wMtHoBji307dsI5drp
VokELweu6SS7M4ODE8/Ci3QLS/mmx++9s2kCCqq49dyA2/ZabLb2nBF96wo/RDp9
3kIzfNvzMkC3VRwESV+SUG0x
-----END CERTIFICATE-----
```

### CA requirements

When using an arbitrary certificate and key for a `provided` backend, we must make sure that we comply with the following requirements:

1. It MUST have basic constraint `CA` set to `true` (see [X509-SVID: 4.1. Basic Constraints](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#41-basic-constraints))
2. It MUST have key usage extension `keyCertSign` set (see [X509-SVID: 4.3. Key Usage](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#43-key-usage))
3. It MUST NOT have key usage extension 'keyAgreement' set (see [X509-SVID: Appendix A. X.509 Field Reference](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#appendix-a-x509-field-reference))
4. It SHOULD NOT set key usage extension 'digitalSignature' and 'keyEncipherment' to be SPIFFE compliant (see [X509-SVID: Appendix A. X.509 Field Reference](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md#appendix-a-x509-field-reference))

{% warning %}
Do not use the following example in production, instead generate valid and compliant certificates. This example is intended for usage in a development environment.
{% endwarning %}

Below we can find an example to generate a sample CA certificate + key:

{% tabs %}
{% tab openssl %}

The following command will generate a CA root certificate and key that can be uploaded to {{site.mesh_product_name}} as a Secret and then used in a `provided` mTLS backend:

```sh
SAMPLE_CA_CONFIG="
[req]
distinguished_name=dn
[ dn ]
[ ext ]
basicConstraints=CA:TRUE,pathlen:0
keyUsage=keyCertSign
"

openssl req -config <(echo "$SAMPLE_CA_CONFIG") -new -newkey rsa:2048 -nodes \
  -subj "/CN=Hello" -x509 -extensions ext -keyout key.pem -out crt.pem
```

The command will generate a certificate at `crt.pem` and the key at `key.pem`. We can generate the {{site.mesh_product_name}} Secret resources by following the [Secret resource](/docs/{{ page.release }}/production/secure-deployment/secrets/).

{% endtab %}
{% endtabs %}

### Development Mode

In development mode we may want to provide the `cert` and `key` properties of the `provided` backend without necessarily having to create a Secret resource, but by using an inline value.

{% warning %}
Using the `inline` modes in production presents a security risk since it makes the values of our CA root certificate and key more easily accessible from a malicious actor. We highly recommend using `inline` only in development mode.
{% endwarning %}

{{site.mesh_product_name}} offers an alternative way to specify the CA root certificate and key:

{% tabs %}
{% tab Kubernetes %}

Please note the `inline` properties that are being used instead of `secret`:

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
      - name: ca-1
        type: provided
        conf:
          cert:
            inline: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHekNDQWdPZ0F3S... # cert in Base64
          key:
            inline: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHekNDQWdPZ0F3S... # key in Base64
```

{% endtab %}

{% tab Universal %}

Please note the `inline` properties that are being used instead of `secret`:

```yaml
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: provided
      conf:
        cert:
          inline: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHekNDQWdPZ0F3S... # cert in Base64
        key:
          inline: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURHekNDQWdPZ0F3S... # cert in Base64
```

{% endtab %}
{% endtabs %}

## Permissive mTLS

In version 1.4.1 and later, {{site.mesh_product_name}} provides `PERMISSIVE` mTLS mode to let you migrate existing workloads with zero downtime.

Permissive mTLS mode encrypts outbound connections the same way as strict mTLS mode, but inbound connections on the server-side
accept both TLS and plaintext. This lets you migrate servers to an mTLS mesh before their clients. It also supports the case
where the client and server already implement TLS.

{% warning %}
PERMISSIVE mode is not secure. It's intended as a temporary utility. Make sure to set to `STRICT` mode after migration is complete.
{% endwarning %}

{% tabs %}

{% tab Kubernetes %}

```yaml
apiVersion: kuma.io/v1alpha1
kind: Mesh
metadata:
  name: default
spec:
  mtls:
    enabledBackend: ca-1
    backends:
      - name: ca-1
        type: builtin
        mode: PERMISSIVE # supported values: STRICT, PERMISSIVE
```

{% endtab %}

{% tab Universal %}

```yaml
type: Mesh
name: default
mtls:
  enabledBackend: ca-1
  backends:
    - name: ca-1
      type: builtin
      mode: PERMISSIVE # supported values: STRICT, PERMISSIVE
```

{% endtab %}

{% endtabs %}

## Certificate Rotation

Once a CA backend has been configured, {{site.mesh_product_name}} will utilize the CA root certificate and key to automatically provision a certificate for every data plane proxy that it connects to `kuma-cp`.

Unlike the CA certificate, the data plane proxy certificates are not permanently stored anywhere but they only reside in memory. These certificates are designed to be short-lived and rotated often by {{site.mesh_product_name}}.

By default, the expiration time of a data plane proxy certificate is `30` days. {{site.mesh_product_name}} rotates these certificates automatically after 4/5 of the certificate validity time (ie: for the default `30` days expiration, that would be every `24` days).

You can update the duration of the data plane proxy certificates by updating the `dpCert` property on every available mTLS backend.

You can inspect the certificate rotation statistics by executing the following command (supported on both Kubernetes and Universal):

{% tabs %}
{% tab kumactl %}

We can use the {{site.mesh_product_name}} CLI:

```sh
kumactl inspect dataplanes
# MESH      NAME     TAGS          STATUS   LAST CONNECTED AGO   LAST UPDATED AGO   TOTAL UPDATES   TOTAL ERRORS   CERT REGENERATED AGO   CERT EXPIRATION       CERT REGENERATIONS
# default   web-01   service=web   Online   5s                   3s                 4               0              3s                     2020-05-11 16:01:34   2
```

Please note the `CERT REGENERATED AGO`, `CERT EXPIRATION`, `CERT REGENERATIONS` columns.

{% endtab %}
{% tab certificate-rotation HTTP API %}

We can use the {{site.mesh_product_name}} HTTP API by retrieving the [Dataplane Insight](/docs/{{ page.release }}/reference/http-api#dataplane-overviews) resource and inspecting the `dataplaneInsight` object.

```json
...
dataplaneInsight": {
  ...
  "mTLS": {
    "certificateExpirationTime": "2020-05-14T20:15:23Z",
    "lastCertificateRegeneration": "2020-05-13T20:15:23.994549539Z",
    "certificateRegenerations": 1
  }
}
...
```

{% endtab %}
{% endtabs %}

A new data plane proxy certificate is automatically generated when:

- A data plane proxy is restarted.
- The control plane is restarted.
- The data plane proxy connects to a new control plane.
