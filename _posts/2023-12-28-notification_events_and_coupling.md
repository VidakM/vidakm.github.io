---
layout: post
title: 🛍️ Notification events and strong coupling
date: 2023-12-28 10:05:00
description: What are the hazards of Notification events and the tight coupling they create and when should it be used?
tags: events event-driven microservices data
categories: guidelines
thumbnail: assets/blog_images/notification_events_and_coupling/banner_notification_events_and_coupling.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/notification_events_and_coupling/banner_notification_events_and_coupling.png" class="img-fluid rounded z-depth-1" %}

As we mentioned in previous articles, there are two types of Fact Events, notifications, and state transfer events. Notification events are lightweight Fact Events that don’t carry much information, but only tell you an action occurred, and which entities were involved. State transfer events on the other hand contain the complete delta of changes, or the full result model after the change has occurred. 

Here we will discuss the nature of notification events and what their use cases are. To understand the details of state transfer events you can read the [( :flags:CQRS and state in Event Driven Services)](/blog/2023/cqrs_and_state/) article. State events are typically used to implement business logic, orchestrate processes, or synchronize data across different components or systems.

### Use cases
They are typically used to report status changes, errors, warnings, progress updates, or other short but relevant information.  One use case for notification events is to monitor the health and performance of a system or a service. For example, a notification event can be emitted when a service starts, stops, crashes, or recovers from a failure. These events can be consumed by other components that are interested in the service's status, such as logging systems, dashboards, alerting systems, or recovery mechanisms.

Another use case for notification events is to provide feedback to users or clients about the outcome of their actions or requests. For example, a notification event can be emitted when a user submits a form, uploads a file, completes a purchase, or receives a message. These events can be consumed by other UI components through SignalR that are responsible for displaying the feedback to the user, such as user interfaces, notifications systems, or email services. [(:satellite:BFFs and Event Driven Design / How to solve eventual consistency in UI)](/blog/2023/bffs_and_event_driven/#how-to-solve-eventual-consistency-in-ui)

### Strong coupling hazard
When services subscribe to notification events, they may need more data than what the event provides to perform their tasks. This can create a dependency between services and reduce the system's performance, as each event may trigger a query request for more information. This goes against the main benefits of using events and event driven design, which are to achieve service decoupling and performance isolation. If an entire system is built using this pattern, cascading failures may also happen at any point as any service can unknowingly become crucial.

{% drawio path="assets/blog_images/notification_events_and_coupling/Untitled_Diagram-1681211688011.drawio.xml" page_number=0> height=200 %}

### GDPR and necessary use case
In certain cases, due to regulations or constraints, state events must be downscaled to notification events. For example, due to GDPR, persisted events are not allowed to store PII, so they are downscaled to only include the persona ID instead. Constraints like these can lead to tight coupling between services but might be necessary.

We want sensitive information to always be structured, clearly labeled, and responsibly stored, thus it cannot be littered all over the network. Event Logging solutions do provide the ability to encrypt stored events with Crypto Shredding, but it’s difficult to guarantee all event consumers will abide. By restricting sensitive data access to APIs only, we can authenticate access only to consumers treating the data responsibly. 

In conclusion, for sensitive data it’s a good approach to store PII and sensitive data locally in their service, only emit them in notification event and then provide good APIs for querying and bulk operations that a consumer might need. This will require the service to be scaled to handle requests and have redundancies in place to avoid cascading failures.

{% drawio path="assets/blog_images/notification_events_and_coupling/Untitled_Diagram-1682066949840.drawio.xml" page_number=0> height=600 %}

## References

Internal:

* [( :flags:CQRS and state in Event Driven Services)](/blog/2023/cqrs_and_state/)
* [(:satellite:BFFs and Event Driven Design / How to solve eventual consistency in UI)](/blog/2023/bffs_and_event_driven/#how-to-solve-eventual-consistency-in-ui)

