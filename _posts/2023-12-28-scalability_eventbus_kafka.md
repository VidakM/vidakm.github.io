---
layout: post
title: 🧮 Scalability and Event Bus (Kafka)
date: 2023-12-28 12:38:00
description: What is an eventbus and what are the simplest TLDR scalability advice for maintaining throughput while never skipping a beat
tags: events event-driven microservices eventbus data environment
categories: guidelines
thumbnail: assets/blog_images/scalability_eventbus_kafka/banner_scalability_eventbus_kafka.jpg
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/scalability_eventbus_kafka/banner_scalability_eventbus_kafka.jpg" class="img-fluid rounded z-depth-1" %}

Kafka is one of many event buses, sometimes also referred to as event streams or event grids. They are systems enabling one-to-many message publishing, queuing, and in-order delivery, to groups of consumers. They are helpful as they enable asynchronous or delayed processing, as opposed to API solutions such as REST or gRPC, which are blocking and require immediate responses with threats of timeout. They are also very fault tolerant, and help multiple consumers keep track of where they last left off and enable a simple interface to resume and load balance consumption. If one consumer fails, another one can continue where it left off.

They are the backbone of Event Driven Architecture, as they help Event Driven Services observe and keep a historical record of the world around them. They are based on eventual consistency, the notion that there's no “now” state in a perpetually evolving distributed system and that we instead should build systems that will eventually receive all events and settle in the correct state. Doing atomic CRUD is doomed to be complex.  Read more about them here [(:mailbox_with_no_mail: Event Driven Services)](/blog/2023/event_driven_services/) and about eventual consistency here [(:hourglass_flowing_sand:CRUD in Microservices)](/blog/2023/CRUD_in_microservices/) .

### Broker and Topics
An event bus is at its core a distributed message broker, which enables publishers to push events to “topics” that subscribers can consume from. Subscribers to do so by using a broker client, that connects to the broker and pulls down messages. A topic can be created by a producer or by the configuration in a broker. They are defined simply by their string name, such as io.contoso.reportingservice.events but can however have configurations and have more properties. To read more on naming conventions, look here [( :mailbox_with_no_mail:Event Driven Services | Event bus and topics)](/blog/2023/event_driven_services/#event-bus-and-topics). To understand events read this [( :e-mail:Events )](/blog/2023/events/) .


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683297116253.drawio.xml" page_number=0> height=300 %}

### Topics and loose couplings
The key aspect here is that the producer and consumers are disconnected and unaware of each other. They only share the knowledge of the topic's existence and perhaps the format of events emitted through it. This enables systems to expand with new services and features without affecting producers. This leads to greater autonomy and looser coupling, as services don’t need to know each other’s DNS records, REST endpoints or similar.

By designing in this fashion, we can quickly expand our systems with many services doing a variety of things. All we need to do is to know which topics to subscribe to for the data we need, and to ensure we version our events in a way that will enable simpler maintenance. You can read more about that here [(:card_box:Event Versioning and Upgrades)](/blog/2023/event_versioning/). 

### Partitions and scaling
As some producers will produce a lot more than others, and some consumers will need to parallelize more work than others, we an effective way to increase our throughput. To do so, topics are further divided into partitions. A partition can be seen as a hidden topic within a topic that the even bus uses to dynamically distribute load between consumers. It splits up the events into multiple hidden queues and so enables many instances of the same service (consumer groups) to consume events concurrently, instead of working with slow Mutexes and locking conditions on a single queue.

Partitions can be assigned to dedicated brokers running on dedicated VMs or nodes, further ensuring event bus avilability and fault tolerance. 

{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683298156642.drawio.xml" page_number=0> height=400 %}

Topics and their partitions can only be scaled up, but never down. This is because we can always create one more partition, but merging multiple partitions, while having consumers and competing locks is very difficult and would have a major hit on performance. Most event buses, such as Kafka, don’t support this operation and instead recommend creating a new topic.

### Partition Ordering
Messages in a single partition are guaranteed to arrive in order. So, if it is crucial that events on a topic are processed in order of publish, it can be good to have a single partition. For example, a `service.events` topic so that consumers can consume all Fact Events in order. 

Otherwise, you can use the `Kafka Message Key header` to ensure events with the same key always end up in the same partition. This can for example be the Tenant Id, so that you can get all events regarding a tenant in a single partition, for in-order processing. Receiving a `tenant.updated` event before ever seeing the `tenant.created` event, due to messages being out of order, would be a hazard or exception.  This can also be useful if running globally distributed, messages with a German tenants key would be stored in the German partition, even if pubished in US.

As soon as a new consumer is added or removed, the broker tries to rebalance and distribute messages from various partitions more fairly to increase throughput by scaling horizontally.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683206980630.drawio.xml" page_number=0> height=400 %}

However, it is worth pointing out that Kafka can only parallelize and scale out a single topic to the number of partitions that exist. So, if you have 3 partitions, and deploy a 4th consumer, that consumer will sit idle as all partitions are being consumed. This arises as sharing a single partition across consumers causes a lot of locking.

As the partition balancing and head tracking is per consumer group, that means that different services can scale horizontally in an asymmetric fashion, while also keeping the current head at different locations. Example:


### Consumer groups
When consumers start consuming, they subscribe to a topic and either create or join a consumer group. A consumer group is just a string used by different consumers to indicate to the broker they are consuming using a shared head tracked. So that when one instance reads an event, the other instance reads the next event and they can continue where the other left off. Consumer groups can be created by any consumer and are usually just the name of the service.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683207828012.drawio.xml" page_number=0> height=400 %}

### How to consume more
Initially when looking at a topic you will realize that the consumer client only pulls 1 message at a time, and is by default only using a single thread. This might feel counter intuitive as we are not utilizing the full capacity of the CPU.  However, Kafka is designed for microservice use cases and envisions the consumers to parallelize by running many instances instead.

In order to process more than one message at a time, the consumer would need to pull and pre-buffer a few of them locally before processing. However, without acknowladging them to the broker, another consumer might process the same message. And if the pre-buffered and un-processed messages would be acknowledged, there is a chance they might get lost if the consumer crashes for some reason before completing them.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683209191790.drawio.xml" page_number=0> height=400 %}

Acknolwedgment can either be automatic based on time, on successful processing or manual. The Kafka client passes the message to the process and can either automatically acknowledge successful consumption to the broker after a short period of execution without exception, assuming success. It can also wait with acknowledgment until the processing function returns. Or the processing function can run the client in manual mode, deciding itself to acknowledge each message when it sees fit.

By default, the consumer will pick up 1 message, process it util success, and only then will the client ack to the broker and move the consumer group head forward.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683209854278.drawio.xml" page_number=0> height=300 %}


### Do
Thus, we should strive to have networks with very manu small consumers for each topic, so that we can parallelize by instances and not by buffering. Looking something like this.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683211498471.drawio.xml" page_number=0> height=500 %}


### Avoid
We should avoid have large, resource consuming services that try to buffer up messages and deal with manual acknowledgments.


{% drawio path="assets/blog_images/scalability_eventbus_kafka/Untitled_Diagram-1683209972001.drawio.xml" page_number=0> height=500 %}
 
Don’t drop state in consution. publish with ack, publish without ack, publish async with callback ack.

Difficult to scale by batching. DLQ and other things such as filiure queue.

Scale by having more consumers instead, works with stateless microservices.

No more consumers than partitions.
 