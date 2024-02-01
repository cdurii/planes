---
title: Welcome to Kuma
---

Welcome to the official documentation for Kuma, a modern distributed **Control Plane** with a bundled Envoy Proxy integration.

The word "Kuma" means "bear" in Japanese (クマ).

<center>
<img src="/assets/images/diagrams/main-diagram@2x.png" alt="" style="width: 450px; padding-top: 10px"/>
</center>

The core maintainer of Kuma is **Kong**, the maker of the popular open-source Kong Gateway 🦍.

## Get started

{% if_version lte:2.1.x %}
[Read about service mesh](/docs/{{ page.version }}/introduction/what-is-a-service-mesh)

[Read about Kuma](/docs/{{ page.version }}/introduction/what-is-kuma)

[Install Kuma](/install/latest/)

[Jump to quickstart](/docs/{{ page.version }}/quickstart/kubernetes/)

[Explore the API](/docs/{{ page.version }}/reference/http-api)
{% endif_version %}
{% if_version gte:2.2.x %}
[Read about service mesh](/docs/{{ page.version }}/introduction/about-service-meshes)

[Read about Kuma](/docs/{{ page.version }}/introduction/overview-of-kuma)

[Install Kuma](/install/latest/)

{% if_version gte:2.6.x %}[Jump to quickstart](/docs/{{ page.version }}/quickstart/kubernetes-demo/){% endif_version %}{% if_version lte:2.5.x %}[Jump to quickstart](/docs/{{ page.version }}/quickstart/kubernetes/){% endif_version %}

[Explore the API](/docs/{{ page.version }}/reference/http-api)
{% endif_version %}
