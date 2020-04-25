# Front-end reference architecture

*Author(s)*: Kevin Pennekamp | front-end architect | [kevtiq.co](https://kevtiq.co) | <hello@kevtiq.co>

> This document describes a front-end reference architecture for digital enterprises. It offers a framework-agnostic architecture focused on the application behind the user interface.

## Introduction
This reference architecture aims to enable front-end engineers to create large scale applications for enterprises. These applications are characterized by large code-bases, long development time and. many external connections. But most importantly, many users. To achieve control over the business outcomes (adaptability, predictability, quality and innovation) of the application, three principles are key:

- **Scalability**, to deal with new user-features, new external APIs the application has to connect to and more heavy background tasks;
- **Maintainability**, by applying  a solid structure, responsibilities of tasks, [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), and modularity in the user interface;
- **Resilience**, to ensure a stable user experience, by applying safe-guards in the heart of the application. 

In the remainder of this document, several diagrams are displayed. The meaning of the different types of blocks in these diagrams are specified in the legend, below.

![](/images/architecture-legend.png)

## High-level overview
The main idea behind the reference architecture is to implement [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). To facilitate this, a simple [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) with three layers is introduced:

- The **routing** layer is part of the presentation layer. This layers determines which module(s) are presented to the user;
- The **modules** layer represents the remainder of the presentation layer, but also the business layer. The majority of the work will be in this layer. A module (or a [cell](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md) of modules) is created by applying [domain driven development](https://martinfowler.com/bliki/BoundedContext.html) in this layer;
- The **core** layer represents the application and data access layer.

![](/images/architecture-high-level.png)

## Application core layer
The core layer of this reference architecture consists of several linked components, as visualized and detailed below. Combined, these components contribute to three principles of this architecture. 

- An application **store** that holds application critical data. This data directly impacts how the application behaves for users;
- An **gateway** that is responsible for all outgoing communication. In case of a single external source, a single API client suffices. But when there are multiple external sources, the gateway acts like a mediator/facade that manages all requests for the different sources; 
- The **pub/sub** is used to synchronize other components in the core layer, and asynchronously update the presentation layer when updates (e.g. responses from an API) come in. In addition, it can be used to allow for cross-browser tab synchronization of critical data; 
- A **process manager** that is used to prioritize and manage heavy operations that run in the background on various (web-)workers.

![](/images/architecture-core.png)

Besides these linked components, several other components can live in the core layer. Examples are the browser **history** stack, or a **system tracker** that can be used for error handling and logging. 

### Application store

Access layer is responsible that the correct events are send to the pub/sub in case of nested updates.

![](/images/architecture-core-store.png)

State management should following the event-sourcing pattern (e.g. [Redux-style guide](https://redux.js.org/style-guide/style-guide) of [GactJS](https://github.com/gactjs/store/blob/master/docs/white-paper.md)). Event sourcing requires immutability, serializability (cloneable and reference-agnostic), and state centralization. There are two ways to build up the centralized state for an application:

- Use reducer functions that handle events (e.g. Redux). These reducers determine how the state is mutating based on the provided information. Biggest downside is that the reducers have to be configured in one place, making the application state configuration highly coupled;
	```
	reducer: (state, action) => state
	```
- Use a decoupled access layer that allows for `set`, `get`, `update` and `remove` events. This allows decoupled useage of the global state.  

In case the global state manages complex flows, [statecharts](https://statecharts.github.io/) should be used to shape (a part of) the state. Statecharts is another implementation of event-based state management. When using reducers, you can put in guards to mock a statechart. In case of an access layer, the actual shaping of the new state (through a statechart) happens outside of the state. A new state value is created and the `set` or `update` injects this into the global state.

### API Gateway

> **NOTE**: when there is only one external source, only one API client is used. Many API client packages (e.g. [Apollo Client](https://www.apollographql.com/client/) implement several of the described concepts and flows. 

There can be multiple API clients in the application at the same time (REST, SOAP, WebSocket, GraphQL, etc.). By placing a `mediator` on top of these clients, the UI components and other core parts of the application only have to interact with a single ’client’ object, the facade. 

Some of these API clients can have their own caching mechanism (e.g. Apollo Client), but most don’t. By using the ‘Proxy’ pattern, an internal cache can be build into the `mediator` on top of all API clients. This cache should be according the `state-while-revalidate` pattern, which basically uses a set period in which a cache item is deemed valid. On every cache update, this period is set anew. This makes it also possible to combine data from different sources in the cache. One could also use the application state for this.

The `mediator` could include a ‘circuitbreaker’ that becomes into effect when one API server errors (e.g. 404).

The `mediator` could also be used to refresh authentication information (e.g. JWT tokens). Instead of checking this in the middleware of each client, it can be managed in the `mediator`, removing the need of a connection between the API clients and the stores. 

![](/images/architecture-core-gateway.png)

The `mediator` could apply middlewares for each outgoing request. This middleware could, for instance, be used for authentication token refreshing. The middleware checks if the authentication information is still valid.

If not, the request is aborted, and a callback is subscribed to the ‘refresh_completed’ event in the pubsub. If no refresh request is started (which is stored in the application state), a refresh request is started. On completion, the ‘refresh_completed’ event is fired on the pubsub.

Each API client can have its own middleware. Adding for instance a `Authentication Bearer` to a request object can differ per client. Some require middleware to do it (Apollo Client) while others require it to be in the request function itself as metadata (Axios). This means that adding the information to a request cannot happen in the middleware of the `mediator`. 

## Modules

> DDD

### Module architecture

![](/images/architecture-module.png)

actions vs. [saga pattern](https://microservices.io/patterns/data/saga.html)

### Types of modules

### Example

> Example twitter response

## Components

![](/images/architecture-component.png)


