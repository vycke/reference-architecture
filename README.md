# Front-end reference architecture

*Author(s)*: Kevin Pennekamp | front-end architect | [kevtiq.co](https://kevtiq.co) | <hello@kevtiq.co>

This document describes a front-end reference architecture for digital enterprises. It offers a framework-agnostic architectural best practices focused on the application behind the user interface.

## Introduction
The goal of this architecture is to enable front-end engineers to create large scale applications for enterprises. These applications are characterized by large code-bases, long development time and. many external connections. But most importantly, many users. To achieve control over the business outcomes (adaptability, predictability, quality and innovation) of the application, three principles are key:

- **Scalability**, to deal with new user-features, new external APIs the application has to connect to and more heavy background tasks.
- **Maintainability**, by applying  a solid structure, responsibilities of tasks, [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), and modularity in the user interface.
- **Resilience**, to ensure a stable user experience, by applying safe-guards in the heart of the application. 

In the remainder of this document, several diagrams are displayed. The meaning of the different types of blocks in these diagrams are specified in the legend, below.

![](images/architecture-legend.png)

## High-level overview
The main idea behind the reference architecture is to implement [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). To facilitate this, a simple [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) with three layers is introduced:

- The **routing** layer is part of the presentation layer. This layers determines which module(s) are presented to the user.
- The **modules** layer represents the remainder of the presentation layer, but also the business layer. The majority of the work will be in this layer. A module (or a [cell](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md) of modules) is created by applying [domain driven development](https://martinfowler.com/bliki/BoundedContext.html) in this layer.
- The **core** layer represents the application and data access layer.

![](images/architecture-high-level.png)

## Application core
The core layer centralizes critical components, as visualized below. This centralization contributes to the maintainability of the application. Some components use a 'mediator' to allow for a `n` number of child components. The application core holds the following components: 

- An application **store** that holds application critical data. This data directly impacts how the application behaves for users.
- An **gateway** that is responsible for all outgoing communication. This can be as single API client, or a mediator with routing requests towards multiple external sources.
- The **[pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)** is used to synchronize other components in the core layer, and asynchronously update the presentation layer. In addition, it can be used to allow for cross-browser tab synchronization of critical data; 
- A **process manager** that mediates and prioritizes computational heavy operations that run in the background on various (web-)workers.

![](images/architecture-core.png)

> **NOTE**: Not all components are required at all times. A process manager is only required when working with (multiple) (web-)workers. 

Besides these components, several other components can live in the core layer. Examples are the browser **history** stack, or a **system tracker** that can be used for error handling and logging. 

### Application store
An application store, or data storage, is used for global state management, and is often required for large-scale front-end applications. Ideally, the application store follows the patterns around [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html). This means that the store should be: 

- The data is stored in a **centralized** place.
- **Event driven** to ensure that the store at determines how the data should change, based on the event.
- **Immutable** to avoid the data in the store being mutated from outside of the store, increasing the resilience of the application.

To comply with the principles of this architecture, an **access layer** that decouples the state interface, is used. This allows for higher scalability and maintainability. Store events (`get`, `set`, `update`, `remove` or `transaction`) can be defined and invoked on a module-level. The access layer handles these events and applies them on the **data storage**.

![](images/architecture-core-store.png)

> **NOTE**: many front-end applications use global state managements for all data. Many existing global state management packages like [Redux](https://redux.js.org/style-guide/style-guide) have a coupled state interface. Although state events can be defined elsewhere, they have to be configured in the store nonetheless.

Whenever a component (from the core layer, or a UI component) triggers an event, the data storage is changed. The access layer also sends an event (including the changed data) via the pub/sub (except in case of a `get` event). Other components can subscribe to these events and act whenever the data changes. If the data structure in the storage is tree-like, the access layer is responsible to ensure that the corresponding tree-paths are provided with events via the pub/sub.

### API Gateway

> **NOTE**: when there is only one external source, only one API client is used. Many API client packages (e.g. [Apollo Client](https://www.apollographql.com/client/) implement several of the described concepts and flows.

Complex application often require connection to multiple external sources. These sources can require different types of connections (e.g. REST and GraphQL). To comply with the principles of the reference architecture, a [**mediator**](https://en.wikipedia.org/wiki/Mediator_pattern) is used. Within the mediator, each external source gets its own dedicated **client**. 

![](images/architecture-core-gateway.png)

Each request, regardless of the related external source, goes through the mediator. The mediator sends each request though three components:

- The gateway **cache** is a proxy that stores all responses for various request definitions, for a certain period of time ('state-while-revalidate' pattern).
- A [**circuit breaker**](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) maintains the state of the external source. If a server error is received, outgoing requests are bounced to prevent reoccurring failure.
- Each request is enhanced or aborted by a chain of **middleware** (e.g. the refreshing of authentication information before the actual request is send out). Each middleware in the chain has access to the application store and it can subscribe to store events via the pub/sub.

After a request goes through these blocks, it will go to its specific client. The response is handled by passing it back to the component sending the request. In case of a `cache-network` strategy, the values from the cache are directly provided back to the component sending the request. In the mean time, the request goes through the chain and is send to the source. The moment the mediator receives the response, the mediator sends the response over the pub/sub. As the requesting component is subscribed to the pub/sub, it can update the shown cached values with the new values. 

## Modules

Domain driven development

### Module architecture

![](images/architecture-module.png)

### Types of modules

### Example

## Components

![](images/architecture-component.png)


