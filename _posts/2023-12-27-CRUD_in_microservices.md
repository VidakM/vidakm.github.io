---
layout: distill
title: ⏳ CRUD in microservices
date:   2023-12-27 09:05:00
description: How can we handle transactions in distributed systems and what are the major pitfalls?
tags: events event-driven services data
categories: guidelines
thumbnail: assets/blog_images/crud_in_microservices/banner_crud_in_microservices.png
authors:
  - name:  Vidak Mijailovic
    url: "https://www.threads.net/@vidmij" 
    affiliations:
      name: KTH, Netigate
---

{% include figure.html path="assets/blog_images/crud_in_microservices/banner_crud_in_microservices.png" class="img-fluid rounded z-depth-1" %}

CRUD (create, read, update, delete) services, with REST or gRPC operations work very well for isolated datasets. This means datasets which are only affected by or composed in an isolated and predictable environment. For example, within a service that can perform atomic operations on multiple SQL tables. However, cross service CRUD consistency is difficult.

Cross service CRUD consistency is difficult for several reasons, but primarily as there are no join operations across datasets, no normalization, no single system image and no single system state. As the network can fail, transaction guarantees are weak, especially if going across three or more services and they require work implementing consistency. Often times failiure managment ends up spreading to business logic. Two-phase commit transactions and other distributed networking patterns for point-to-point communication are complex.

(TBD, write)  explain a two phased commit here [Two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)


{% drawio path="assets/blog_images/crud_in_microservices/Untitled_Diagram-1681817317091.drawio.xml" page_number=0> height=500 %}
 

### Strong consistency is the wrong default expectation in distributed systems.

Information has latency, information is always from the past. Sometimes more and sometimes less, but all information is from the past and it’s in the eye of the beholder to interpret it. The news is always relative, and we all experience a different present. This should be embraced, there is no “now”.
We should expect and rely on eventual consistency, as distributed systems have an inherited latency and failure rate. The fact that information will have disturbances and arrive later should be accounted for and expected, not fought. This is how reality works, and we should embrace the constraints of reality. 
Distributed systems are non-deterministic with failures, lost messages, interruptions. Exceptions happen all the time. It is thus the space between microservices that can help us warp and enable the perspective of the services to see the correct reality and eventually arrive at the right outcomes. Building a service is simple, dealing with the space between them, the sea of the network cables, is the hard part. 

>“In a system which cannot count on distributed transactions, the management of uncertainty must be implemented in the business logic” - [Pat Helland](https://ics.uci.edu/~cs223/papers/cidr07p15.pdf)

Thus, we should make sure the beholder always ends up seeing the truth.

> “An autonomous component can only promise its own behavior… Autonomy makes information local, leading to greater certainty and stability” - [Mark Burgess](https://www.amazon.com/Search-Certainty-science-information-infrastructure/dp/1492389161)

Events [(:e-mail:Events)](/blog/2023/events/) and Event Driven Services [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/) can lead to greater certainty in the system and decrease business logic impact. Events can help us craft autonomous islands of determinism which are always up to date and eventually correct. To ensure consistency between services, we need a good ability to ensure the past always catches up with each other and have a good protocol for which events you accept and which you emit. 

### How should we then view data and the space between services?
Pat Helland:
 - Inside each service is the present, the current state and truth.  
 - Outside each service is the past, the stream of events coming to affect our perception of reality. The event horizon is coming closer.
 - Between services, hope is for the future, commands. Hoping that someone will listen to them.

Microservices are a never-ending stream of convergence, always trying to catch up. The system is always in motion, there is no “now” in a distributed system. Thus, we should strive to perform as many transactions and operations as possible using persistent truths. This can be addressed using Event Driven Services [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/)

{% drawio path="assets/blog_images/crud_in_microservices/Untitled_Diagram-1681816453804.drawio.xml" page_number=0> height=530 %}

### Managing failure with Event Driven Services

Events can help manage failure instead of trying to avoid it with patterns, try-catch statements, backoff protocols etc. They are unavoidable so we should make sure we can manage it instead of avoiding it. Even during an app deployment, a similar state can be reached when gracefully shutting down.

A strongly coupled failure management system such as those found in monoliths can be avoided by following these principles instead. 

Failures need to be contained within the services to avoid cascading failures. They should be captured as events and asynchronously notify any interested listener. The command emitter should be able to notice this event and cancel or handle own operation, if necessary, in a reverse Saga pattern [(:mailbox_with_no_mail:Event Driven Services / Example use cases SAGA)](/blog/2023/event_driven_services/#example-use-cases---saga). The service itself could listen to the failure event as a listen-to-yourself-pattern (TBD, write) and place it on a job queue for later handling. This can then be managed by other services in the system just like any other Fact Event.

{% drawio path="assets/blog_images/crud_in_microservices/Untitled_Diagram-1681817529617.drawio.xml" page_number=0> height=530 %}


## References

Internal:

* [(:e-mail:Events)](/blog/2023/events/)
* [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/)
* [(:mailbox_with_no_mail:Event Driven Services / Example use cases SAGA)](/blog/2023/event_driven_services/#example-use-cases---saga)


External:
* [Two-phase commit protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
* [Pat Helland](https://ics.uci.edu/~cs223/papers/cidr07p15.pdf)
* [Mark Burgess](https://www.amazon.com/Search-Certainty-science-information-infrastructure/dp/1492389161)
* [Designing Events-First Microservices - Jonas Boner - QCon 2018](https://www.youtube.com/watch?v%253D1hwuWmMNT4c)
* [The Many Meanings of Event-Driven Architecture - Martin Fowler GOTO 2017](https://www.youtube.com/watch?v%253DSTKCRSUsyP0%2526list%253DPLnWKhEdO_Yk0glJySVV6NC9G0NgnhW4UW%2526index%253D1)