---
layout: post
title: 🌐 Regions, domains, and URL formats
date: 2023-12-28 13:00:00
description: What are event driven servies and why should we use them?
tags: events event-driven microservices
categories: guidelines
thumbnail: assets/blog_images/regions_domains_dns/banner_regions_domains_dns.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/regions_domains_dns/banner_regions_domains_dns.png" class="img-fluid rounded z-depth-1" %}

Composing URLs for a multi cloud and premise deployment strategy can be complex as there are many aspects to consider. Here we will walk through a few common ones and propose alternatives to solving them.
* How to structure an app URL to work across hybrid deployments and shifting customer base.
* Segment public facing and private app APIs.
* Exposing product URLs with clear namespaces and objectives.

### Subdomain per tenant - for complex deployments
There are frequent shifts in needs for data and customer hosting and we need a good strategy for dealing with them. Regulations such as GDPR and Schrems II are posing new requirements on data retention and storage and there are more regulations developing. In tandem, customer needs are constantly shifting and in some segments such as fintech and insurance, we need a flexible strategy for managing access and hosting, even enabling hybrid scenarios such as partial on-premises or multi-cloud. 

By enabling customers to access their apps using a subdomain as prefix to our public URL, we can hide a lot of the complexity and enable hybrid scenarios while also simplifying our ingress and monitoring strategy. A customer can sign into our master URL and get proxied to their SaaS or on-premises solution depending on package used. Example: CX product might be SaaS based while the EX is on-premises.

This also simplifies global access, as we can partition customers by region and datacenter and route a specific subdomain to the nearest datacenter hosting customer data. Anycast and other DNS technologies can aid us here with a transparent interface.

Furthermore, as each customer could have a dedicated subdomain ingress, we can scale and tier them independently while also simplifying monitoring. Customers would be differenciated in each request path, simplifying tracing and ingress monitoring by eliminating the need to lookup headers, GUIDs or tokens.  

Proposal: `<customer>.<company>.<tld>`
Example: `contoso.netigate.io`

{% drawio path="assets/blog_images/regions_domains_dns/Untitled_Diagram-1681915329535.drawio.xml" page_number=0> height=300 %}

### API endpoints - Apps and product vs customer APIs

#### UI app (:bento:Microfrontends )
All clients should be served through a dedicated path to accommodate micro-frontends. A user should be able to reach a domain and browse a multitude of clients without noticing they are leaving the domain. For example, `company.io/persona`, then going to `company.io/settings` and finally `company.io/reporting`. This is key to micro frontends. (Microfrontends architecture)

Proposal for UI: `<customer>.<company>.<tld>/<client-service-name>` 

Example: `contoso.netigate.io/reports`
 
#### BFF (:satellite:BFFs and Event Driven Design)
All BFF APIs should be using the domain with `/api` as path prefix followed by their `service-name`, combined into `/api/<service-name>`.  Preferably, their `service-name` is similar or same to their `client-service-name`. 

Proposal for BFF: `<customer>.<company>.<tld>/api/<service-name>`

Example: `contoso.netigate.io/api/reports`


If a BFF is expected to have high throughput and is a general service endpoint, they might deserve a dedicated ingress or even their own subdomain. An example of this could be a generic survey response BFF serving a survey response app.

Proposal for BFF: `<endpoint-service-name>.<company>.<tld>`

Example: `response.netigate.io`
 
#### Customer API 
Customer APIs, whose purpose is to enable customer integration as means of selling value should be separated from BFF APIs. As these services are not BFFs but backend services that expose APIs directly to customers, they can have varying loads and throughput and often place strains on scalability in different patterns from Apps and BFFs. It can be lucrative to combine multiple BFFs into a single Customer API too, which should be avoided as usage from one category might affect the others. We want the ability to scale the app and Customer API independently.

Customer APIs are often deployed to dedicated clusters and often have dedicated read-only DB replicas following :flags:CQRS and state in Event Driven Services.

As they are generic endpoints for integrations, they are not segmented per customer but instead directly use API as subdomains. Customers are identified using tokens. This also eliminates confusion if an API is used internally, externally or both.

Proposal for customer API: `api.<company>.<tld>` 
Examples from current production: `api.netigate.io`


{% drawio path="assets/blog_images/regions_domains_dns/Untitled_Diagram-1681914830875.drawio.xml" page_number=0> height=300 %}

#### Product URL - for independent teams 

As the product portfolio grows it is common to deploy multiple products over time which also need to scale independently and have their own ingresses. This might not just be necessary for the global ingress, but also for the backend ingress, as each product or service might be hosted independently. As the we have discussed in this article - Microfrontends architecture - you might even want to split up a single app into multiple views.  

Thus, we need the ability to differentiate common components between their product domains. For example, both the EX and CX product might have a Response Service, which has the same purpose, but very different implementations tailored to specific domains. 

To keep service names clear it is good to differentiate the URL based on the domain or namespace when deploying to production.

Proposal: `<customer>.<company>.<tld>/api/<namespace>/<service>`

Example EX: `contoso.netigate.io/api/ex/response`
Example CX: `contoso.netigate.io/api/cx/response`

Proposal for UI: `<customer>.<company>.<tld>/<namespace>/<client-service-name>`

Example: `contoso.netigate.io/ex/reports`

### Summary example for customer Contoso:
Showing a customer Contoso accessing the EX-module and browsing the distribution settings.

Customer portal for login `company.io/login` with BFF auth from `company.io/api/auth`. After auth, forwarding to app : `contoso.company.io/ex/app` served from BFF at `contoso.company.io/api/ex/app`. Customer API for integrations is dedicated at: `api.company.io`

### Multi-cloud and Event Driven Service
As services can communicate asynchronously through events using for example Aiven Kafka at scale, events can be used to decouple services and orchestrate complex workflows across different domains and clouds.

By using events, services do not need to know the location or availability of other services, which eliminates the need for a Service Mesh, Service Discovery and advanced DNS configurations. Event Driven Service design with Kafka and Events can enable simple cross cloud architectures that are scalable, resilient and adaptable.

{% drawio path="assets/blog_images/regions_domains_dns/Untitled_Diagram-1681813875423.drawio.xml" page_number=0> height=500 %}