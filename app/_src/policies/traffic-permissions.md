---
title: Traffic Permissions
---
{% if_version gte:2.6.x %}
{% warning %}
New to {{site.mesh_product_name}}? Don't use this, check the [`MeshTrafficPermission` policy](/docs/{{ page.release }}/policies/meshtrafficpermission) instead.
{% endwarning %}
{% endif_version %}


{% tip %}
Traffic Permissions is an inbound policy. Dataplanes whose configuration is modified are in the `destinations` matcher.
{% endtip %}

This policy provides access control rules to define the traffic that is allowed within the [Mesh](/docs/{{ page.release }}/production/mesh/). 

Traffic permissions requires [Mutual TLS](/docs/{{ page.release }}/policies/mutual-tls) enabled on the [Mesh](/docs/{{ page.release }}/production/mesh/). Mutual TLS is required for {{site.mesh_product_name}} to validate the service identity with data plane proxy certificates. If Mutual TLS is disabled, {{site.mesh_product_name}} allows all service traffic. 

{% if_version gte:2.6.x %}
Since {{site.mesh_product_name}} version 2.6.x, the control plane no longer creates a default `TrafficPermission`. The default `TrafficPermission` allows all communication between all services in the new `Mesh`. Make sure to configure your policies to allow appropriate access to each of the services in your mesh.
{% endif_version %}
{% if_version lte:2.5.x %}
The default `TrafficPermission` policy that {{site.mesh_product_name}} creates when you install allows all communication between all services in the new `Mesh`. Make sure to configure your policies to allow appropriate access to each of the services in your mesh.
{% endif_version %}

As of version 1.2.0, traffic permissions support the `ExternalService` resource. This lets you configure access control for traffic to services outside the mesh.

## Usage

To specify which source services can consume which destination services, provide the appropriate values for `kuma.io/service`. This value is required for sources and destinations.

{% tip %}
**Match all**: You can match any value of a tag by using `*` -- for example, like `version: '*'`.
{% endtip %}

{% tabs %}
{% tab Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: allow-all-traffic
spec:
  sources:
    - match:
        kuma.io/service: '*'
  destinations:
    - match:
        kuma.io/service: '*'
```
Apply the configuration with `kubectl apply -f [..]`.
{% endtab %}
{% tab Universal %}
```yaml
type: TrafficPermission
name: allow-all-traffic
mesh: default
sources:
  - match:
      kuma.io/service: '*'
destinations:
  - match:
      kuma.io/service: '*'
```
Apply the configuration with `kumactl apply -f [..]` or with the [HTTP API](/docs/{{ page.release }}/reference/http-api).
{% endtab %}
{% endtabs %}

You can use any [Tag](/docs/{{ page.release }}/production/dp-config/dpp/#tags) with the `sources` and `destinations` selectors. This approach supports fine-grained access control that lets you define the right levels of security for your services.

## Access to External Services

The `TrafficPermission` policy can also be used to restrict traffic to [services outside the mesh](/docs/{{ page.release }}/policies/external-services).

### Prerequisites

* {{site.mesh_product_name}} deployed with [transparent proxying](/docs/{{ page.release }}/production/dp-config/transparent-proxying/)
* `Mesh` configured to [disable passthrough mode](/docs/{{ page.release }}/production/mesh/#usage)

These settings lock down traffic to and from the mesh, which means that requests to any unknown destination are not allowed. The mesh can't rely on mTLS, because there is no data plane proxy on the destination side.

### Usage

First, define the `ExternalService` for a service that is not in the mesh.

{% tabs %}
{% tab Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: ExternalService
mesh: default
metadata:
  name: httpbin
spec:
  tags:
    kuma.io/service: httpbin
    kuma.io/protocol: http
  networking:
    address: httpbin.org:443
    tls:
      enabled: true
```
{% endtab %}
{% tab Universal %}
```yaml
type: ExternalService
mesh: default
name: httpbin
tags:
  kuma.io/service: httpbin
  kuma.io/protocol: http
networking:
  address: httpbin.org:443
  tls:
    enabled: true
```
{% endtab %}
{% endtabs %}

Then apply the `TrafficPermission` policy. In the destination section, specify all the tags defined in `ExternalService`.

For example, to enable the traffic from the data plane proxies of service `web` or `backend` to the new `ExternalService`, apply:

{% tabs %}
{% tab Kubernetes %}
```yaml
apiVersion: kuma.io/v1alpha1
kind: TrafficPermission
mesh: default
metadata:
  name: backend-to-httpbin
spec:
  sources:
    - match:
        kuma.io/service: web_default_svc_80
    - match:
        kuma.io/service: backend_default_svc_80
  destinations:
    - match:
        kuma.io/service: httpbin
```
{% endtab %}
{% tab Universal %}
```yaml
type: TrafficPermission
name: backend-to-httpbin
mesh: default
sources:
  - match:
    kuma.io/service: web
  - match:
      kuma.io/service: backend
destinations:
  - match:
      kuma.io/service: httpbin
```
{% endtab %}
{% endtabs %}

Remember, the `ExternalService` follows [the same rules](/docs/{{ page.release }}/policies/how-kuma-chooses-the-right-policy-to-apply) for matching policies as any other service in the mesh -- {{site.mesh_product_name}} selects the most specific `TrafficPermission` for every `ExternalService`.

## All options

{% json_schema TrafficPermission type=proto %}
