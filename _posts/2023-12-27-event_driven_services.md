---
layout: post
title: 📬 Event driven services
date:   2023-12-27 09:00:00
description: What are event driven servies and why should we use them?
tags: events event-driven microservices
categories: guidelines
thumbnail: assets/blog_images/event_driven_services/banner_event_driven_services.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/event_driven_services/banner_event_driven_services.png" class="img-fluid rounded z-depth-1" %}

Event Driven Services are services that primarily operate in a reactive way using Events. They strive to have a clearly defined Bounded Context and keep mutable state within themselves, but externally communicate Fact Events and only act on incoming Command Events or other relevant Fact Events.

This enables developers to write business logic based on Fact Events in the network, or on user Command events in a clearer way that is in line with Domain Driven Design. As all actions are taken based on events, it is simple to implement event logging (Event Sourcing) for replayability and tracing to enable recalculation of state. 

Distributed systems have an inherited latency, and microservice architecture leads to difficulties performing joins. Event Sourcing enables eventual consistency in an architecture, and any corrections and changes are eventually emitted as new Fact Events, the system will always settle in the correct state. This allows downstream components to rely on stable values found in Fact Events and can always confidently act on them. As they know they will be notified when mutations happen, they can always adjust the current state. You can read more here [( :hourglass_flowing_sand:CRUD in Microservices / Strong consistency is the wrong default expectation in distributed systems)](/blog/2023/CRUD_in_microservices/#strong-consistency-is-the-wrong-default-expectation-in-distributed-systems)

As actions are triggered through events, it enables greater parallelism, decoupling, extensions, and resiliency. Because events are managed by a scalable event bus, they can have an unlimited number of subscribers after publishing, whose performance and characteristics don’t affect the publishers and can be scaled independently. Features can be built on top of events downstream without knowledge of producer, making it suitable for other teams to add value. Service failures or crashes don’t result in loss of action as the event can be read by another instance. You can read more here [( :flags:CQRS and state in Event Driven Services )](/blog/2023/cqrs_and_state/)

As Event Driven services often log incoming events locally before acting, it leads to them having a cache of all their dependencies. This can be beneficial as partial network downtime doesn’t lead to an outage, just eventual consistency. The cache also means the service, in case being an API or BFF, can reply to requests directly and confidently without making further calls. It also helps perform complex joins across multiple services using your local cache. [( :flags:CQRS and state in Event Driven Services / How to receive Event Sourcing)](/blog/2023/cqrs_and_state/#how-to-receive---event-sourcing)

BFFs commonly use this pattern [( :satellite:BFFs and Event Driven Design.)](/blog/2023/bffs_and_event_driven/)

This can be summarized in Event Driven Services becoming autonomous islands of determinism. An informative read for the opposite of Event Driven services can be found here in CRUD in Microservices [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/)

General example of services emitting events downstream to trigger further work.


{% drawio path="assets/blog_images/event_driven_services/Untitled_Diagram-1680523402741.drawio.xml" page_number=0> height=500 %}


### Event bus and topics

Events are usually not distributed directly between services but are usually published to an event bus that enables a publish and subscribe distribution method. This enables event distributions to become one-to-many and decouples the publishers from consumers, as the publisher isnt aware of subscribers. Popular event buses are [Apache Kafka](https://kafka.apache.org/), [NATS](https://nats.io/) and [RabbitMQ](https://www.rabbitmq.com/) to mention a few.

Common to all event buses that use the publish and subscribe pattern is that they allow a service to define a topic onto which they publish one or many types of events. Services can then subscribe to interesting topics and gain a persistent stream of events.

Ownership, or declaration, of topics on an eventbus should be based on event owner and event type.

A service should publish all Fact Events on a topic it owns and be the sole publisher to it. This is crucial so that consumers know which types of Fact Events and messages in general they can expect. 

A service should have all its Command Events written to a topic it owns and be the sole subscriber to it. This is crucial so that consumers have a well-defined interface towards the service and so the service is built for all Command Events it might expect.     

A service can also have a single, or set of, private topics used internally by itself. This can be for example as a [Dead Letter Queue](https://en.wikipedia.org/wiki/Dead_letter_queue), or internal retry work topic. This is also a practical way to host a reply topic when implementing the SAGA pattern [(:mailbox_with_no_mail:Event Driven Services / Example use cases SAGA)](/blog/2023/event_driven_services/#example-use-cases---saga) or a data topic when starting up for the first time [(:rocket:Event Driven Service startup first time / Deploying to existing environment)](/blog/2023/event_driven_startup/#deploying-to-existing-environment).

Service Fact Event topic suggestion: `<company>.<namespace>.<service>.events`

Service Command Event topic suggestion: `<company>.<namespace>.<service>.commands`

Service Private Fact Event topic suggestion: `<company>.<namespace>.<service>.private.[*, .reply, .dlq]`

There is a case to be made that a service should be able to have more granular Fact Event topics such as per model. But most event buses, such as Kafka, only guarantee event in-order delivery for messages in a single topic and partition. Thus, it becomes difficult for consumers to build up a correct event log, or simply reating in the correct order, when subscribing to many topics. 


{% drawio path="assets/blog_images/event_driven_services/Untitled_Diagram-1680780636238.drawio.xml" page_number=0> height=500 %}

### Event bus for Pipeline services

Data pipeline services are similar to Event Driven services; however, they are built to chain data between themselves to generate a larger output from a stream of data in a felxible way, while providing resiliency. You can read more about them here [(Building & Operating High-Fidelity Data Streams)](https://www.infoq.com/articles/high-fidelity-data-streams/?utm_source=pocket_reader) (TBD, write).

In short, as each service works towards very specialized and niche results and are built to work together, the next service in the chain wants to optimize its event consumption. As the next service in the pipeline is only interested in the previous step(s) result, it is recommended to produce in a dedicated `service.results` topics beyond the generic `service.events`. 

Pipeline Service Result Fact Event topic suggestion: `<company>.<namespace>.<service>.results`


{% drawio path="assets/blog_images/event_driven_services/Untitled_Diagram-1680782368382.drawio.xml" page_number=0> height=500 %}

### Loose couplings and Contracts

Keep in mind that the services should be loosely coupled and not directly have any knowledge of each other's Fact Events consumers. Thus, a service and its Fact Events should not be tailored to any specific consumer nor be bound to a specific use case. But general and re-usable. Fact Events, just like Command Events, should be considered contracts between producers and consumers that need to be versioned with backwards compatability or slowly deprecated over time after handshake.

Events as contracts is the key to scaling and reliability of Event Driven Services as they keep the promise of services working as independent observers to each other, allowing extensions to the network without disrupting the broader ecosystem. As a service might have an unknown number of consumers, modifying a Fact Event without backwards compatibility, or publishing intentions ahead of time, might have unknown cascading effects on others, leading to a distributed monolith. An event beeing to use cases specific, might also lead to constant team coordination efforts and slowdown. 

### Example use cases - SAGA

Command events can be used to perform actions, but also to retrieve data or perform complex asynchronous transactions. When used to trigger transactions and retrieve or confirm a result, they are usually emitted by a service assigned as “Orchestrator” and it’s called the Saga Pattern.

A worker service can publish a topic and define Command events for use by the network. A client service can then issue a Command Event on the network and wait for a result. The result can either be published on the worker services general event topic, or on a topic the client specifies as a private reply topic in the Command Event. 

Example. Emit event of type `PersonaService.CreatePersona` on the topic `arcadia.personaservice.commands` with property `reply_topic:arcadia.personabff.private.reply` . Wait for command results on `arcadia.personabff.private.reply`. The resulting Command Event should a format specified by the worker service.  Read more here: [( :e-mail:Events / Command event Saga)](/blog/2023/events/#command-event-with-reply---saga)

The benefit of this pattern is resiliency. As the commands are issued through an event bus, a failure in one service will simply result in another service taking over the work. Service outage will result in the commands resuming once up agian. 

The opposite of this pattern is to use direct CRUD point-to-point commands. To read more about them and why they might pose a problem in distributed systems see this [( :hourglass_flowing_sand: CRUD in Microservices)](/blog/2023/CRUD_in_microservices/) .


{% drawio path="assets/blog_images/event_driven_services/Untitled_Diagram-1680524450442.drawio.xml" page_number=0> height=550 %}

### Multi-cloud and Event Driven Service
As services can communicate asynchronously through events using for example Aiven Kafka at scale, events can be used to decouple services and orchestrate complex workflows across different domains and clouds.

By using events, services do not need to know the location or availability of other services, which eliminates the need for a Service Mesh, Service Discovery and advanced DNS configurations. Event Driven Service design with Kafka and Events can enable simple cross cloud architectures that are scalable, resilient and adaptable.

{% drawio path="assets/blog_images/event_driven_services/Untitled_Diagram-1681813875423.drawio.xml" page_number=0> height=500 %}




## References

Internal:

* To understand why to build an Event Driven Service read 
  * [( :hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/)
* CRUD in Microservices and for how to implement an Event Driven Service read 
  * [ :flags: CQRS and state in Event Driven Services](/blog/2023/cqrs_and_state/)
* [( :satellite:BFFs and Event Driven Design.)](/blog/2023/bffs_and_event_driven/)
* [(:mailbox_with_no_mail:Event Driven Services / Example use cases SAGA)](/blog/2023/event_driven_services/#example-use-cases---saga)
* [(:rocket:Event Driven Service startup first time / Deploying to existing environment)](/blog/2023/event_driven_startup/#deploying-to-existing-environment)
* [( :e-mail:Events / Command event Saga)](/blog/2023/events/#command-event-with-reply---saga)



External:
* [Apache Kafka](https://kafka.apache.org)
* [NATS](https://nats.io) 
* [RabbitMQ](https://www.rabbitmq.com) 
* [Dead Letter Queue](https://en.wikipedia.org/wiki/Dead_letter_queue)
* [Building & Operating High-Fidelity Data Streams](https://www.infoq.com/articles/high-fidelity-data-streams/?utm_source=pocket_reader)
* [Designing Events-First Microservices - Jonas Boner - QCon 2018](https://www.youtube.com/watch?v%253D1hwuWmMNT4c)
* [Event Design & Event-first Development - David Boyne GOTO 2022](https://www.youtube.com/watch?v=GBzr48Mj2JU&list=PLnWKhEdO_Yk0glJySVV6NC9G0NgnhW4UW&index=8)
* [The Many Meanings of Event-Driven Architecture - Martin Fowler GOTO 2017](https://www.youtube.com/watch?v%253DSTKCRSUsyP0%2526list%253DPLnWKhEdO_Yk0glJySVV6NC9G0NgnhW4UW%2526index%253D1)