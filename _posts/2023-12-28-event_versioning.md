---
layout: distill
title: 🗃️ Event Versioning and Upgrades
date: 2023-12-28 12:00:00
description: How can we efficiently version and upgrade our events without breaking existing consumers and ensuring replayability into the future?
tags: events event-driven microservices data
categories: guidelines
thumbnail: assets/blog_images/event_versioning/banner_event_versioning.png
authors:
  - name:  Vidak Mijailovic
    url: "https://www.threads.net/@vidmij" 
    affiliations:
      name: KTH, Netigate
---

{% include figure.html path="assets/blog_images/event_versioning/banner_event_versioning.png" class="img-fluid rounded z-depth-1" %}

When working in an Event Driven Architecture it is crucial that we maintain loose couplings between services so that our network can be scaled, extended, and upgraded without constantly needing to retrofit changes. Here we will discuss how to design events in such a way that the loosely coupled and unknown consumers gain resilience and can be independently developed without stopping. For more on Event Driven Services, look here: [(:mailbox_with_no_mail: Event Driven Services)](/blog/2023/event_driven_services/) 

### Event upgrades and contracts
All events are in fact API contracts between services and need to be properly versioned and announced before breaking changes. If changed, they should be semantically versioned and changed in such a way to either be backwards compatible or replaced with a brand new one in case of major shifts. Otherwise, consumers will always have to coordinate and play a catchup game with breaking charges, which is expensive. You can read more about this here: [( :mailbox_with_no_mail:Event Driven Services / Loose couplings and Contracts)](/blog/2023/event_driven_services/#loose-couplings-and-contracts)

### Extending
We should generally be adding to models, not changing, as adding properties does not cause a conflict. As long as we don’t break the previous definition with a new version, we can convert older stored event versions to new versions. If we follow this rule, old consumers can always understand newer versions and we can release producers safely without notifying consumers.

Keeping a compatible historic trace of events leading up to the current state is the pilar of Event Driven Design and Event Logging. In particularly when it comes to fault correction by recalculating the current state. Thus, we need to be able to replay events from the start. Read more about Event Sourcing and Logging here: [(:flags:CQRS and state in Event Driven Services / How to receive Event Sourcing)](/blog/2023/cqrs_and_state/#how-to-receive---event-sourcing)

By following our rule, old versions already stored on disk can be converted or upcast to the new version before transitioning to them and event consumers only needs to know how to handle the latest event. It also enables future processing of old event versions.

During version upgrades, new fields must be either nullable or have default values defined, in some cases we can even manually compose them. For example, if the field “firstname” and “surname” are added based on a past property containing a “fullname”.


{% drawio path="assets/blog_images/event_versioning/Untitled_Diagram-1683205469674.drawio.xml" page_number=0> height=700 %}

### Breaking
If the new event version must contains a breaking change, then this should be considered a brand new event instead and gain its own topic. This event can be emitted side by side with the legacy event for an extended period of time until consumers can upgrade.

### Don’t strongly type serializer
Strongly typing events on an eventbus is not recommended - by for example using a serializer that only maps to exact objects from events via a schema registry. It will result in a strong coupling between producers and consumers, as all changes to the event will require the consumers to be re-compiled with the new event format in mind, causing developer overhead and potentially downtime. 

Consumers will always need to be ready before the deployment of the producer, which leads to a tight coupling and slows the release process as we need to coordinate and implement before release. Changes in API contracts tend to be very expensive in comparison to other changes, as they necessitate changes for all downstream consumers. 
 
{% drawio path="assets/blog_images/event_versioning/Untitled_Diagram-1683190886364.drawio.xml" page_number=0> height=500 %}

### Loosely type and gain resiliance
Instead, we recommend using JSON or preferably Protobuf schema definitions to loosely serialize events into objects. This works by mapping the incoming event to the schema defined to produce an object. Properties matching the same name get used, properties missing in the existing schema simply get ignored, and properties contained in the schema but missing from the event get set to null or their default values. 

This requires us to work with two rules. Nothing can be renamed, if it has to be, then it must become a new event. We cannot change the semantic meaning of a property, as the mapping assumption will cease to work. You can read a lot more on this topic in Grogory Youngs articles: [Read Versioning in an Event Sourced System (Leanpub)](https://leanpub.com/esversioning/read#leanpub-auto-weak-schema)

{% drawio path="assets/blog_images/event_versioning/Untitled_Diagram-1683189966052.drawio.xml" page_number=0> height=350 %}

There are several upsides of this technique. We never need to upgrade or upcast a past even, all old events will always be compatible with the latest version, in the sense that the consumer will not crash. Consumers will always have at least the data points it was built to function with and can upgrade to consume the new additional data at a future point.

 {% drawio path="assets/blog_images/event_versioning/Untitled_Diagram-1683192564081.drawio.xml" page_number=0> height=700 %}
 
 
 
## References

Internal:

* [(:mailbox_with_no_mail: Event Driven Services)](/blog/2023/event_driven_services/) 
* [( :mailbox_with_no_mail:Event Driven Services / Loose couplings and Contracts)](/blog/2023/event_driven_services/#loose-couplings-and-contracts)
* [(:flags:CQRS and state in Event Driven Services / How to receive Event Sourcing)](/blog/2023/cqrs_and_state/#how-to-receive---event-sourcing)


External:
* [Read Versioning in an Event Sourced System (Leanpub) - Grogory Youngs](https://leanpub.com/esversioning/read#leanpub-auto-weak-schema)
