# Front-end reference architecture

**version**: 0.2.1 |
**Author(s)**: Kevin Pennekamp | front-end architect | [kevtiq.dev](https://kevtiq.dev) | <hello@kevtiq.dev>

This document describes a reactive front-end reference architecture for digital enterprises. It offers a framework-agnostic architectural best practices focused on the application behind the user interface.

## Introduction

The goal of this architecture is to enable front-end engineers to create large scale applications for enterprises. These applications are characterized by large code-bases, long development time and many external connections. But most importantly, many users. To achieve control over the business outcomes (adaptability, predictability, quality and innovation) of the application, an [antifragile](https://www.sciencedirect.com/science/article/pii/S1877050916302290) architecture is required, with five key principles:

- **Consistency** in user experience, but also for the engineers working on the application.
- **Resilience**, to ensure a stable user experience, by applying safe-guards in the heart of the application.
- Update user interfaces **reactively** based on interactions and changes in the state.
- **Scalability**, to deal with new features, external sources and heavy background tasks.
- **Maintainability** forced combining the scalability and consistency principles with concepts like [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), and modularity in the user interface.

In the remainder of this document, several diagrams are displayed. The meaning of the different types of blocks in these diagrams are specified in the legend, below.

![](images/architecture-legend.png)

## High-level overview

The main idea behind the reference architecture is to implement [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). To facilitate this, a simple [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) with three layers is introduced:

- The **routing** layer is part of the presentation layer. This layers determines which module(s) are presented to the user.
- The **modules** layer represents the remainder of the presentation layer, but also the business layer. The majority of the work will be in this layer. A module (or a [cell](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md) of modules) is created by applying [domain driven development](https://martinfowler.com/bliki/BoundedContext.html) in this layer.
- The **core** layer represents the application and data access layer.

![](images/architecture-high-level.png)

## Application core

The core layer centralizes critical components, as visualized below. This centralization contributes to the maintainability of the application. Some components use a 'mediator' to allow for a `n` number of child components. The components below can be present in the core layer.

- An application **store** that holds application critical data. This data directly impacts how the application behaves for users.
- An **gateway** that is responsible for all outgoing communication. This can be as single API client, or a mediator with routing requests towards multiple external sources.
- The **[pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)** is used to synchronize other components in the core layer, and asynchronously update the presentation layer. In addition, it can be used to allow for cross-browser tab synchronization of critical data;
- A **process manager** that mediates and prioritizes computational heavy operations that run in the background on various (web-)workers.

![](images/architecture-core.png)

Besides these components, several other components can live in the core layer. Examples are the browser **history** stack, and an **error tracker**.

### Application store

An application store, or data storage, is used for global state management, and is often required for large-scale front-end applications. Ideally, the application store follows the patterns around [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html). This means that the store should be:

- The data is stored in a **centralized** place and is normalized, i.e. relational data should not be nested.
- **Event driven** to ensure that the store at determines how the data should change, based on the event.
- **Immutable** to avoid the data in the store being mutated from outside of the store, increasing the resilience of the application.

To comply with the principles of this architecture, an **access layer** (or [**proxy**](https://en.wikipedia.org/wiki/Proxy_pattern)) that decouples the state interface, is used. This allows for higher scalability and maintainability. Store events (`get`, `set`, `update` or `remove`) can be defined and invoked on a module-level. The access layer handles these events and applies them on the **data storage**.

![](images/architecture-core-store.png)

> **NOTE**: many front-end applications use global state managements for all data. Many existing global state management packages like [Redux](https://redux.js.org/style-guide/style-guide) have a coupled state interface. Although state events can be defined elsewhere, they have to be configured in the store nonetheless.

Whenever a component triggers an event, the data is changed. The access layer sends an event (including the changed data) via the pub/sub (except in case of a `get` event). Other components can subscribe to these events and act whenever the data changes. With normalized data, sending the correct events for related data is made possible, providing a more resilient experience.

### API gateway

> **NOTE**: in case of only one external source, a single API client can replace the gateway. Many open source API clients support most components described in this section (e.g. [Apollo Client](https://www.apollographql.com/client/)).

The API gateway enables the application to connect to multiple external sources (e.g. REST and GraphQL) with dedicated API **clients** in a consistent way. It extracts critical components and let a [**mediator**](https://en.wikipedia.org/wiki/Mediator_pattern) share them with different API clients.

![](images/architecture-core-gateway.png)

Each request, regardless of the related external source, goes through the mediator. The mediator sends each request though three components before it hits the API client:

- The gateway **cache** is a proxy that stores all responses for various request definitions, for a certain period of time ('state-while-revalidate' pattern).
- A [**circuit breaker**](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) maintains the state of the external source. If a server error is received, outgoing requests are bounced to prevent reoccurring failure.
- Each request is enhanced or aborted by a chain of **middleware** (e.g. the refreshing of authentication information). Each middleware in the chain has access to the application store and it can subscribe to store events via the pub/sub.

After the middleware, the correct `client` is chosen by the mediator. After a response is received, the mediator sends it back to the request initiator and the cache. In case of a `cache-network` strategy, the mediator first gives back a value from the cache to the initiator, before the request is send through the middleware and client. After the response is received, the cache is updated and the initiator receives the updated value.

> **NOTE**: in case your chosen UI framework does not allow of UI updates around asynchronous calls, you can let the component subscribe to the pub/sub and have the mediator send the response via the pub/sub. you can utilize the pub/sub.

## Modules

Domain driven development

### Module architecture

![](images/architecture-module.png)

### Types of modules

### Example

## Components

![](images/architecture-component.png)

## CSS & user interface styling
