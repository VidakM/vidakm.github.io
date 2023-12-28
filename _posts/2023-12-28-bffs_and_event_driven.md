---
layout: post
title: 📡 BFFs and Event Driven Design
date:  2023-12-28 09:05:00
description: What are event driven servies and why should we use them?
tags: events event-driven bff front-end
categories: guidelines
thumbnail: assets/blog_images/bffs_and_event_driven/banner_bffs_and_event_driven.png
toc:
  sidebar: left
---

{% include figure.html path="assets/blog_images/bffs_and_event_driven/banner_bffs_and_event_driven.png" class="img-fluid rounded z-depth-1" %}


A BFF (Backend for frontend) is a design pattern that creates a layer between the frontend and the backend of an application. The BFF acts as a proxy that handles requests from the frontend and communicates with the backend services. The BFF can also perform tasks such as authentication, authorization, caching, data transformation, and error handling.

### Why to use a BFF
There are many benefits to having a BFF, here are a few key ones.
* One of the main benefits of using a BFF is that it allows the frontend to have a tailored API that suits its specific needs and preferences. For example, the BFF can compose data from multiple backend services and present it in one single joined model or implement APIs the backend services don’t have. Read more about that here :flags:CQRS and state in Event Driven Services | Read services   Composing multiple services into one 
This can improve velocity, performance, usability, and maintainability of the frontend. 
* Having a BFF abstract away the backend API also decouples the frontend from the backend, making it easier to make sweeping changes in backend without affecting the other.
* It also enables greater connectivity, by not forcing every service or small team to implement REST APIs on their service, they are free to use gRPC, Kafka or any other tool that suits their domain best, the BFF simply converts. In some cases Kafka converter apps need to be written, or SignalR becomes an afterthought.
* As a BFF only serves a single frontend or client type, it allows for detailed performance metrics, fine tuned API access management and security. It can be helpful if you want to separate the API between Android, iOS, and Web clients due to different security requirements or investment in segment. For example keeping a client in maintainance mode functioning while changing backend. 
* It also decreases the scope of the backend services, as they might not need to create client specific APIs and don’t need to support client specific protocols such as REST if they don’t want to.
* As each client only accesses a single BFF, the networks gain a clean access policy and simpel endpoint management, decreasing the need for complex API Gateway setups. This also decreases the number of open connections an ingress needs to manage decreasing cost at scale.

It is key to keep in mind that a BFF should never alter or add business logic, it should simply expose it in a frontend friendly way. All business logic changes should be delegated to backend services as they naturally should propagate to the rest of the solution.

Here is an example of a network before (on left) and after (on right) implementing a BFF.

{% drawio path="assets/blog_images/bffs_and_event_driven/Untitled_Diagram-1680695666997.drawio.xml" page_number=0> height=600 %}

### The hazard with BFFs
BFF patterns also come with some potential hazards or pitfalls that should be considered before implementing it.
* A BFF can introduce duplication of logic and data across different BFFs, which can lead to inconsistency, redundancy, and increased testing effort. This can happen if a BFF doesn’t strictly forward data from backend but starts adding its own mutations or business logic. This is a critical flaw as BFFs should never alter business logic, they should simply expose it.
* Another of the main challenges of using a BFF is the risk of creating tight coupling between the frontend and the backend, which can reduce the reusability, scalability, and maintainability of the services. It’s simple to get started and map in incoming REST API to an equivalent backend REST or gRPC endpoint. But after a while the frontend might get requirements, the backend doesn't support and lead to a vicious cycle of backend always lagging and frontend teams asking for new APIs. 
* Moreover, a BFF can create dependencies and conflicts between different teams and services, which can affect the collaboration, communication, and coordination among them. 
* A BFF directly forwarding requests to the backend services will affect their performance, which can be a hazard if the frontend is heavily trafficked. 

{% drawio path="assets/blog_images/bffs_and_event_driven/Untitled_Diagram-1680765467998.drawio.xml" page_number=0> height=500 %}

Therefore, we should view the BFF as a decoupled application built on top of our backend network with the goal of streamlining it for consumption. As it is an App on the existing solution, and a backend service for the front end, it can implement its own caching, querying logic and other assisting tools that enable frontend use cases without affecting core system business logic.

#### Example
An example of a BFF adding their own logic without affecting business logic could be advanced filtering scenarios. A Persona Service contains a group hierarchy and persona objects with varying properties it and either allows retreival of all objects or specific individuals. For an advanced search and filtering capability, the BFF can cache the users to a search platform like Postgres or OpenSearch and enable advanced filtering for the UI. Of course, this will mean it needs to keep this cache up to date, but there are tools for this as we discussed in Event Driven Services and Event Sourcing. https://netigate.atlassian.net/wiki/spaces/NG/pages/598180204/CQRS+and+state+in+Event+Driven+Services#Read-services---Composing-multiple-services-into-one  

### Solving the hazard by decoupling
To solve the tight API coupling and performance coupling, we can use the concepts of Event Sourcing and Event Driven Design. If the BFF would decrease the amount of data retrieval it makes from the backend it would be able to function even if the backend was down, it’s requests would be returned faster, and high user activity would not affect the core system performance (the backend).

You might initially try to cache data locally in the BFF for some requests, but you’ll quickly realize it's difficult to determine if cache misses are due to data genuinely not existing or due to the cache not being up to date.

Thus, you might as well just opt in to building in an event driven model such as an Event Driven Service (https://netigate.atlassian.net/wiki/spaces/NG/pages/597950465 ) from the start and save time in converting later. For implementation guidelines look here https://netigate.atlassian.net/wiki/spaces/NG/pages/598180204. But your output would look like something below.

BFFs would subscribe to Fact Events from other Event Driven Services in the network and build up a local cache using Event Logging and Event Sourcing. All reads would be contained locally to the service. Incoming write requests would be converted to Command Events or performed as direct gRPC requests to ensure direct feedback, for example to get a status code as return.


{% drawio path="assets/blog_images/bffs_and_event_driven/Untitled_Diagram-1680700655916.drawio.xml" page_number=0> height=600 %}

### How to solve eventual consistency in UI
If the BFF is implemented with Event Sourcing and eventual consistency principals mentioned in the previous chapter, it might lead to the UI basing its rendering on globaly outdated but locally correct data. This might be acceptable for many application types, but some might need the UI to always be up to date as the event horizon approaches. 

To adress this we can use the same event driven techniques that Event Driven Services use for the frontend. Event Driven UI is a design pattern that updates the user interface based on events from the server, rather than polling or refreshing the page.

SignalR can help solve this problem by using WebSockets or other fallback techniques to establish a persistent connection between the server and the client. The server can then push events to the client whenever there is a change in the data, such as an update, a deletion, or an insertion, due to an incoming Fact Event. The client can then update the UI accordingly by directly subscribing to Fact Events, without waiting for the BFF's cache to be refreshed and poll again. 
 
 