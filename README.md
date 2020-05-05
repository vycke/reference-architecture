# Front-end reference architecture

##### _version 0.4.0_

**Author(s)**: Kevin Pennekamp | front-end architect | [kevtiq.dev](https://kevtiq.dev) | <hello@kevtiq.dev>

This document describes a reactive front-end reference architecture for digital enterprises. It offers a framework-agnostic architectural best practices focused on the application behind the user interface.

## Introduction

The goal of this architecture is to enable front-end engineers to create large scale applications for enterprises. These applications are characterized by large code-bases, long development time and many external connections. But most importantly, many users. To achieve control over the business outcomes (adaptability, predictability, quality and innovation) of the application, an [antifragile](https://www.sciencedirect.com/science/article/pii/S1877050916302290) architecture is required, with six key principles:

- **Consistency** in user experience, but also for the engineers working on the application.
- **Resilience**, to ensure a stable user experience, by applying safe-guards in the heart of the application.
- The architecture should enable **agility** for developers implement new and changed features, by focusing on reusable UI components.
- Update user interfaces **reactively** based on interactions and changes in the state.
- **Scalability**, by creating multi purpose elements to deal with new features, external sources and heavy background tasks.
- **Maintainability** forced combining the scalability and consistency principles with concepts like [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), and modularity in the user interface.

The architecture goes three levels deeps. The diagrams in the remainder of this document visualize these levels and how they interact with other levels. The meaning of the different types of blocks in these diagrams are specified in the legend, below.

![](images/architecture-legend.png)

## High-level overview

The main idea behind the reference architecture is to implement [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). To facilitate this, a simple [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) with three layers is introduced:

- The **routing** layer is part of the presentation layer. This layers determines which module(s) are presented to the user.
- The **modules** layer represents the remainder of the presentation layer, but also the business layer. The majority of the work will be in this layer. A module (or a [cell](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md) of modules) is created by applying [domain driven development](https://martinfowler.com/bliki/BoundedContext.html) in this layer.
- The **core** layer represents the application and data access layer.

![](images/architecture-high-level.png)

## Application core

The core layer centralizes critical *blocks*, as visualized below. This centralization contributes to the maintainability of the application. The books below can be present in the core layer.

- An application **store** that holds application critical data. This data directly impacts how the application behaves for users.
- An **gateway** that is responsible for all outgoing communication. This can be as single API client, or a mediator with routing requests towards multiple external sources.
- The **[pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern)** is used to synchronize other components in the core layer, and asynchronously update the presentation layer. In addition, it can be used to allow for cross-browser tab synchronization of critical data;
- A **process manager** that mediates and prioritizes computational heavy operations that run in the background on various (web-)workers.

![](images/architecture-core.png)

Besides these blocks, several other blocks can live in the core layer. Examples are the browser **history** stack, and an **error tracker**.

### Application store

An application store, or data storage, is used for global state management, and is often required for large-scale front-end applications. Ideally, the application store follows the patterns around [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html). This means that the store should be:

- The data is stored in a **centralized** place and is normalized, i.e. relational data should not be nested.
- **Event driven** to ensure that the store at determines how the data should change, based on the event.
- **Immutable** to avoid the data in the store being mutated from outside of the store, increasing the resilience of the application.

To comply with the principles of this architecture, an **access layer** (or [**proxy**](https://en.wikipedia.org/wiki/Proxy_pattern)) *element* that decouples the state interface, is used. This allows for higher scalability and maintainability. Store events (`get`, `set`, `update` or `remove`) can be defined and invoked on a module-level. The access layer handles these events and applies them on the **data storage**.

![](images/architecture-core-store.png)

> **NOTE**: many front-end applications use global state managements for all data. Many existing global state management packages like [Redux](https://redux.js.org/style-guide/style-guide) have a coupled state interface. Although state events can be defined elsewhere, they have to be configured in the store nonetheless.

Whenever an element triggers an event, the data is changed. The access layer sends an event (including the changed data) via the pub/sub (except in case of a `get` event). Other elements can subscribe to these events and act whenever the data changes. With normalized data, sending the correct events for related data is made possible, providing a more resilient experience.

### API gateway

> **NOTE**: in case of only one external source, a single API client can replace the gateway. Many open source API clients support most elements described in this section (e.g. [Apollo Client](https://www.apollographql.com/client/)).

The API gateway enables the application to connect to multiple external sources (e.g. REST and GraphQL) with dedicated API **clients** in a consistent way. It extracts critical elements to one place, and let a [**mediator**](https://en.wikipedia.org/wiki/Mediator_pattern) share them with different API clients.

![](images/architecture-core-gateway.png)

Each request, regardless of the related external source, goes through the mediator. The mediator sends each request though three *elements* before it hits the API client:

- The gateway **cache** is a proxy that stores all responses for various request definitions, for a certain period of time ('state-while-revalidate' pattern). The mediator acts as the *data access layer* of the cache, similar as in the application store.
- A [**circuit breaker**](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) maintains the state of the external source. If a server error is received, outgoing requests are bounced to prevent reoccurring failure.
- Each request is enhanced or aborted by a chain of **middleware** (e.g. the refreshing of authentication information). Each middleware in the chain has access to the application store and it can subscribe to store events via the pub/sub.

After the middleware, the correct `client` is chosen by the mediator. After a response is received, the mediator sends it back to the request initiator and the cache. In case of a `cache-network` strategy, the mediator first gives back a value from the cache to the initiator, before the request is send through the middleware and client. After the response is received, the cache is updated and the initiator receives the updated value.

> **NOTE**: in case your chosen UI framework does not allow of UI updates around asynchronous calls, you can let the element subscribe to the pub/sub and have the mediator send the response via the pub/sub. you can utilize the pub/sub.

To ensure resilience, each request should follow the same [statechart](https://statecharts.github.io/), as shown below. When all requests, regardless of their source, follows the same pattern, the API client and/or UI can consistently handle them. Each request starts in _idle_. When a request is started, it moves into the _loading_ state. From this state, four events can happen: success, abort, error or start. In case of the latter, the previous request is aborted and a new request is started.

![](images/architecture-core-gateway-statechart.png)

## Modules

To facilitate [domain driven development](https://martinfowler.com/bliki/BoundedContext.html) around business context, loosely coupled modules are used. A module groups business-related logic, state and UI components. There are three different types of modules identified in this reference architecture. Regardless of their types, modules can be used in a similar way as components, i.e. they can be nested.s

- A **gateway** module functions as a wrapper around other modules, based on routing. It provides logic and state towards the nested modules (e.g. the main application can be seen as a gateway module). 
- Many applications have multiple pages related to each other. **Section** modules combine these pages and their logic into a module (e.g. CRUD pages for user management or authentication page). 
- Sometimes only a single component, or a collection of components are required throughout the application. These are **block** modules (e.g. shopping cart). 

> **NOTE**: these module types are not exclusive. a _gateway_ can also be a _section_, and a _section_ can also be a *block*.

### Module architecture

The module architecture is inspired by the [flux pattern](https://facebook.github.io/flux/docs/in-depth-overview/). It includes several *blocks*, as visualized below. Each module has **components** and **actions**. They represent the view and the logic of a (business-related) module. Both have the ability to interact with the application core. Components can read from the core, while actions can invoke events in the core and await their responses.  

![](images/architecture-module.png)

Besides the components and actions, a module can have a store. It acts similarly to the application store, but is dedicated to the module. Not every module requires a store. The store in a module is often used for modeling business-logic, and have the UI correspond to that. If this is the case, it is recommended that it is shaped like a [state-machine](https://statecharts.github.io/what-is-a-state-machine.html) or [statechart](https://statecharts.github.io/what-is-a-statechart.html). 

> **NOTE**: the store can be implemented in a similar way as the application store, i.e. a data access layer, or features from a framework are used (e.g. React Context) 

there are many different ways to implement the module store. One could create a wrapper component (e.g. React Context) and store the state there. Another implementation could be similar to the application store, including an access layer. 

Components and actions are requirements for all types of modules. However, when working on a gateway or section module, more blocks are required. 

- The **routing** is required when the module is a gateway or section. It determines which **page** or module is shown to the user.
- **Pages** are, in effect, components that are associated with a specific route. It is used in a section module. 

### User interface components
User interface components, or UI components, are the most important parts of the application, and the place you will spend the most time in. It is where the user actually sees and interacts with the application. It consists of five different elements that interact with each other. 

![](images/architecture-component.png)

The API is how a component interacts with its parent, another UI component. The parent component can provide values, configuration and callbacks through the API. The values and configuration are, combined with the component state, used to render the UI. 

A user interacts with the UI. Based on the interaction, an action is invoked. Actions that are single purpose (only used by this component) are defined in the component. But in most cases, actions can and are shared between components. In this case the actions are defined in the module. The action can update the component state or invoke a callback received through the API.

> **NOTE**: the observer described below is often handled by the internals of UI frameworks like React or Vue

A component also has an observer. The observer listens to the values from the API and the state for changes. When a change happens, it invokes a rerender of the UI, and optionally invokes an action. 

