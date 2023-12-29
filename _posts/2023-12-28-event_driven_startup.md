---
layout: post
title: 🚀 Event Driven Service startup first time
date: 2023-12-28 12:18:00
description: What are event driven servies and why should we use them?
tags: events event-driven microservices
categories: guidelines
thumbnail: assets/blog_images/event_driven_startup/banner_event_driven_startup.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/event_driven_startup/banner_event_driven_startup.png" class="img-fluid rounded z-depth-1" %}

When basing your service architecture on Event Driven Services, you strive to keep the amount of CRUD operations low and have the services be fully reactive. However, depending on the environment, this can be difficult to achieve when deploying a service for the first time. [(:mailbox_with_no_mail: Event Driven Services)](/blog/2023/event_driven_services/) 

A few startup problems resolve themselves. Sometimes the service has no events to consume due to the Event Bus not being available yet. The service can also be the first service to be deployed which means it will produce nothing until its dependency services start producing events. However, there are a few more complicated scenarios to consider.

## Deploying to existing environment
Sometimes services need to catch up on past data to perform analysis or track all records from a past point. For example, a reporting service that needs to render reports not only from this month, but from last year's data too. Or an engagement model that needs to calculate web traffic development over time.  This can be difficult if deployed far along into an environment's lifecycle, where the persisted events on the eventbus might already be expiring. 

Most event buses are configured to persist events for 30 days, after which they are pruned. If you are in a mature environment, your local producers of interest might provide an API to re-emit all their event-log, in order, on a private topic for you. This way you can catch up by listening to their event log. But if services have run for many years, this too can be a very time-consuming task.

It might be a good strategy to implement a snapshotting read API in your services to allow quick catchup. On startup, download the entire aggregated model snapshot and then start consuming events to keep up. You can read about event logs, snapshots and CQRS here: [( :flags: CQRS and state in Event Driven Services)](/blog/2023/cqrs_and_state/).

This too can be a time-consuming operation as the datasets might be gigabytes in size. The data retrieval calls could be split up into batches based on dimensions to parallelize, such as per tenant and some other attribute. This could also be further split up or streamed using gRPC Streaming or an eventbus or by asking the producer to publish to one of your private topics. 

{% drawio path="assets/blog_images/event_driven_startup/Untitled_Diagram-1682345915215.drawio.xml" page_number=0> height=600 %}

### After snapshot startup
If your service is expected to have generated data or records for past events too, you should consider emitting Fact Events as you are consuming the past data snapshot. An example might be if you emit a new report for every month of user activity. If users expect to see it retroactively the service should emit it for parts of the past dataset too.

This can, however, be very time-consuming and quite a heavy operation on the event bus too, so consider if it is necessary for your type of service and your potential customers.

### Creation of default objects
Some apps need to provide calculated data if a tenant is present and has made some actions, and default data if otherwise. An example of this can be a product categorization service that has a set of default categories defined for all tenants, but then allows mutations and additions. In case a tenant has never made a mutation, it should return default categories. But in case the service has never been notified a tenant has even be created, what should it do?

Following Event Driven principles, during the startup process and consumption of past events, the service should make an entry for this customer and create the default objects every tenant is expected to have. Regardless of if the tenant has made a request yet or not, an event driven service should be pre-cached and an observing party in the network.

Another example can be the EEX-Engagementservice, serving a list of default Drivers. Or a Survey Library service, serving a list of default templates. 


{% drawio path="assets/blog_images/event_driven_startup/Untitled_Diagram-1682347877003.drawio.xml" page_number=0> height=500 %}


## References

Internal:
* [:mailbox_with_no_mail: Event Driven Services](/blog/2023/event_driven_services/)
* [ :flags: CQRS and state in Event Driven Services](/blog/2023/cqrs_and_state/)
