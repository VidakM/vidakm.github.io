---
layout: post
title: 🎏 CQRS and state in Event Driven Services
date:   2023-12-28 09:10:00
description: What are event driven servies and why should we use them?
tags: events event-driven microservices
categories: guidelines
thumbnail: assets/blog_images/cqrs_and_state/banner_cqrs_and_state.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/cqrs_and_state/banner_cqrs_and_state.png" class="img-fluid rounded z-depth-1" %}

You can use CRUD together with Event Streams (in an event bus) to get an internally consistent materialized view. In a previous chapter [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/) we discussed the difficulties of getting a snapshot of time and a clear aggregated view of a distributed model. Here we will introduce a method for performing joins in a distributed system between disconnected domains, even developed by separate teams.

We will then also introduce Event Sourcing and CQRS as a solution to improve application tracing, model managment, scalability, and event emission.

## Read services - Composing multiple services into one

One way to achieve such a feat is to use Fact Events from both services and aggregate under a new service, from service A and B to C. By doing so, the new service can merge the two datasets into a single composed view and optimize it for consumers. To achieve this, it is crucial that service A stores a record in its database and at the same time, in an atomic way, publishes the data on the Event Bus as a Fact Event. So that all data written to the service is also replicated on the event bus.

This would enable the composition of services A and B into a service C, and for it to act as a complete unit. You can then use service C to query over both datasets and domains in a way that suits their needs but perform writes to the individual services owning the data.

As we have already discussed in CRUD for Microservice, this will lead to a slight delay between a write and reflection in a read, but it will eventually appear correctly. Still, this method is a lot simpler to maintain than a [two-phase-commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol). Achieving atomicity in a distributed system is impossible by nature and the tools discussed below will help us deal with this. For further reading on eventual consistency and CRUD read [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/).

Other benefits are in providng more resilience and decoupling in the system, as extensive reads will not bring down the write system. And the write system being down will not affect reading consumers. 

{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680606519385.drawio.xml" page_number=0> height=600 %}

## How to emit - Outbox pattern  
The outbox pattern is a method to simplify code and make sure all written data becomes emitted as a Fact Event eventually without having circuit breakers and other failsafe mechanisms. Instead of first writing to Database, trying to emit an event, and then potentially dealing with a rollback due to the event bus being down, we can use this pattern.

When writing data to a table, use a transaction to also write a Fact Event to an outbox table. An outbox table is a table of all Fact Events that that the service should emit in order. As both writes are part of a transaction, if one fails it is crucial that the other also fails, so that Fact Events are also created in the correct order. This is how we ensure consistency in all our consumer services. Think of service C above, you wouldn't like to receive an object updated event before seeing the object creation event.

The outbox table can be read and consumed by a seperate async component of the service, such as the Event Publisher in the diagram below. As microservice best practices state a service should own its own database to ensure autonomy and decoupling, we wouldnt recommend having the Event Publisher as its own process. But if you are working on a legacy monolith, or a database without clear owners, then deploying something like a Change Data Capture system can be a way. 

[Change Data Capture (CDC): The Complete Introduction / Confluent](https://www.confluent.io/learn/change-data-capture/)

[What is change data capture (CDC)? - SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16)


{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680607216264.drawio.xml" page_number=0> height=600 %}


## How to receive - Event Sourcing
However, even if following this approach, updates are destructive. When mutating an object to a new version, we are destroying and throwing away the previous value. This data could still be valuable to the business, but its loss can also be difficult to deal with. For example, if the change is caused by a bug, the only way to correct it is to restore a database.

Event logging can help here by applying basic accounting principles. The basic principle is storing incoming events as they come in in order and build up a store ledger. The key is to store events whether you react to them or not, as long as you consume them and deem them potentially relevant. This can be turned into Event Sourcing, a solution for composing models from change events, or just Fact Events.

Event Sourced services work by logging down every incoming event, those that you act on and those that you don’t act on. For the relevant events that carry models interesting to your service, you create a snapshot functionality, keeping an aggregate of a model up to date with the latest change.

### Example
Imagine you are working on a bank account service. You receive an event of a $10 deposit, then a $3 withdrawal event. Taking a snapshot here shows your account balance is $7. Then you keep on logging events the next time users trigger a Fact Event and you maintain the current balance as a snapshot. If a past transaction from a third-party provider didn't reach us in time, it gets added as a new event and the snapshot is recalcualted. Similarly, if a past event was erroneous, upon receiving a correction event we will update the snapshot to corrected state. Not to mention having full history auditing.

{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680675492367.drawio.xml" page_number=0> height=300 %}

Now in case of failure, bug or simply change in business logic, we can reiterate our Event Log and calculate a new correct state. This simplifies database upgrades too because we have all the data locally and don’t have to, for example, perform remote joins. This also enables services to always have a local cache for remote data, enabling them to function independently even when the neighbor is down and in turn not affect neighbors during heavy traffic.

This allows us to have one sole source of truth for all history leading up to the current moment. As we have all data locally, we can continue to function independently and we can always recreate our current state and self-correct.  

This also means we can combine with the Read Service pattern and store events for other domains locally, enabling us to create powerful joined models for our own use cases. This can be useful if you need to perform complex querying of a service not supporting your use cases.  This can be helpful to a BFF needing to cache data.

## Use Case Example external

TBD, link and explain with Marten [https://martendb.io](https://martendb.io) as example

## Create Fact Events - Simplify with CQRS 
As we have explored before in Event Driven Services [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/)  we strive to emit Fact Events as a solution to inherited problems with latency and networking in distributed systems [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/).

By using an internal Event Sourcing mechanism, we can simplify our applications model for work in an event driven environment by allowing Fact Events to naturally materialize into an object. Focusing on making simple delta writes and letting snapshots determin our model states is a great start.

We can expand this further and think of the emitted Fact Event after each model change. If we ensured that each model operation was either a Command (write) or a Query (read) we could simplify our system and not need an ORM. We could then also treat the Event Sourcing write log as our Outbox directly, and not have a separat table for conversion.

{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680611730585.drawio.xml" page_number=0> height=500 %}
 
Splitting up model write and read operations like this is known as CQRS, Command and Query Responsibility Segregation. The key here is to make sure a function either reads or modifies a model, but never does both at the same time. It is a good practice that can be exapnded to any function written in for any type of repository.

It allows others to more easily subscribe to your state changes, as your model is the same as the event.
Besides removing complex mutation code and giving us Fact Events out of the box, it also helps us deal with contention and locks which are the biggest performance killer. As we are always only appending to the write table there is almost never any locking and indexes are very simple. As we have a dedicated read table there is almost no need for locks, models can be optimized and indexes finetuned. Even multiple read views can be created for each model. 

CQRS allows us to store the write model in one way, and the read model in a different more streamlined way. The write log can actaully be a set of events, while the read objects can be snapshots of events 
 
{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680611086333.drawio.xml" page_number=0> height=500 %}
 
## Putting it together
Event Sourcing and CQRS repositories can be compiled into an [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/) that deals with most data destructive issue found in [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/). 

A service can subscribe to many interesting events and log them down before materializing them into usable models with Event Sourcing. Based on business logic it can then issue new Commands to its own repository and mutate its internal state. Because the mutations are clean write Commands, the CQRS repository write log can directly be used as an Outbox pattern and emitted as Fact Events. 
 
{% drawio path="assets/blog_images/cqrs_and_state/Untitled_Diagram-1680612042634.drawio.xml" page_number=0> height=500 %}

 
## References

Internal:

* [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/)
* [(:mailbox_with_no_mail:Event Driven Services)](/blog/2023/event_driven_services/) 



External:
* [two-phase-commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)
* [Change Data Capture (CDC): The Complete Introduction / Confluent](https://www.confluent.io/learn/change-data-capture/)
* [What is change data capture (CDC)? - SQL Server](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server?view=sql-server-ver16)
* [https://martendb.io](https://martendb.io)
