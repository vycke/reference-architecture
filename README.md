# Front-end reference architecture

##### _version 2.0.0_

**Author(s)**: Kevin Pennekamp | front-end architect | [vycke.dev](https://vycke.dev) | <hello@vycke.dev>

This document describes a reactive reference architecture for front-end applications (e.g. single-page applications) on a digital enterprise scale. It offers framework-agnostic best practices focused on the architecture behind the user interface.

## Introduction

The goal of the architecture is to enable engineers to create large-scale applications. These applications have many users, external connections, and long development time. To achieve control over the business outcomes, it requires an [antifragile](https://www.sciencedirect.com/science/article/pii/S1877050916302290) architecture. There are three key principles:

- **Resilience**, to ensure a stable and measurable user experience.
- A **reactive** application that updates based on (user interaction).
- **Composability** that enable development _agility_, resulting in a _scalable_ and _maintainable_ solution with a _consistent_ user experience.

The architecture uses the [C4 architecture](https://c4model.com) notation. The legend below describes the meaning of the different visualizations in this document.

/images/c4-architecture-legend.png

## System context and container overview

The main idea behind the reference architecture is to implement [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). A [layered architecture](https://en.wikipedia.org/wiki/Multitier_architecture) facilitates this by introducing three containers:

- The **router** is part of the presentation layer. This layer determines which modules the user can interact with.
- Each application consists out of many **module** (or [cell](https://github.com/wso2/reference-architecture/blob/master/reference-architecture-cell-based.md)) containers. These represent the presentation layer, but also the business layer. Most of the work will be in these containers. A module represents a part of the business.
- The **core** layer represents the application and data access layer.

/images/c4-architecture-level-2.png

## The core container

The core container centralizes critical _components_, as visualized below. This centralization contributes to the resilience of the application. Combined, there are three important component _groups_ that can be identified in the core container: a **store**, **pub/sub**, and **gateway**.

/images/c4-architecture-level-3-core.png

Besides these components, several other components can live in the core container. Examples are the browser **history** stack, an **error tracker** or a **process manager** mediates and prioritizes heavy background operations (i.e. web-workers) to increase the performance of the web application.

### Core container: store

Large applications use the store for global state management. The recommendation is that the store follows the patterns around [event sourcing](https://martinfowler.com/eaaDev/EventSourcing.html). This means that the store should be:

- It stores data in a **centralized** and normalizes the data, i.e. nesting of relational data is not allowed.
- It is the owner of the data shape and mutations to increase resilience, i.e. it is **event-driven** and **immutable**.

To follow the principles of this architecture, it uses an **access layer**. This _component_ decouples the state interface, allowing for better composability. Store events (`get`, `set`, `update`, or `remove`) can be defined and invoked on a module-level. The access layer handles these events and applies them to the **data storage**, as visualized in the _dynamic diagram_ shown below.

/images/c4-architecture-dynamic-diagram-store.png

> **NOTE**: many front-end applications use global state management for all data. Many existing global state management packages like [Redux](https://redux.js.org/style-guide/style-guide) have a coupled state interface. Although events are defined elsewhere, they have to be configured in the store.

Whenever a component triggers an event, the data is changed. The access layer sends an event (including the changed data) via the pub/sub (except in case of a `get` event). Other components can subscribe to these events and act whenever the data changes. The access layer can also subscribe to the pub/sub. This enables another way for the store to update (e.g. when requests come from a different browser tab).

Data in the data storage is normalized or shaped like a [Statecharts](https://statecharts.github.io/). This allows for more accurate update events to the subscribers.

### Core container: pub/sub
The [pub/sub](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern) adds the ability to synchronize other containers/components on asynchronous tasks. It can update the presentation layer (e.g. UI components in modules) based on updates in the _store_. Another example is to use it for background tasks like synchronization between browser tabs, or resetting an auto-sign out timer.

### Core container: API gateway

> **NOTE**: in case of only one external source, a single API client can replace the gateway. Many open-source API clients support a similar structure (e.g. [Apollo Client](https://www.apollographql.com/client/)).

The API gateway enables a consistent way to connect various external sources or APIs (e.g. REST and GraphQL). The *gateway*, *middleware*, and *client* components act as the API gateway. Each external source has its own corresponding **client** component. This component sends out the actual request. Each client has a chain of **middleware**, that enhances each request (e.g. add authentication information).

/images/c4-architecture-dynamic-diagram-gateway.png

Each request, regardless of the related external source, first goes through the gateway. The gateway is a [**mediator**](https://en.wikipedia.org/wiki/Mediator_pattern) allows for the sharing of generic logic between different clients of different external sources. This mediator handles:

- Bouncing requests based on [RBAC rules](#application-governance).
- Interact with a cache, by getting and setting data. In most cases, this is the application store, but sometimes a separate proxy cache is used. The gateway sets the lifespan of the cached data, using 'state-while-revalidate' pattern.
- [**circuit breaking**](https://en.wikipedia.org/wiki/Circuit_breaker_design_pattern) to ensure only external APIs are called when they are available. If it receives a server error, it bounces future outgoing requests for a limited time, allowing the API to restart itself.
- Implement logic for authentication information refreshing. If a refresh request is in flight, it queues all other requests until the refresh request is finished.
- Send requests to the correct middleware and client.

If a request has a `cache-network` strategy, a cached value from the store is provided first. When the gateway receives the response, it sends the updated data to the request initiator and the cache.

> **NOTE**: in case your chosen UI framework does not allow of UI updates around asynchronous calls, you can let the element subscribe to the pub/sub and have the mediator send the response via the pub/sub. you can use the pub/sub.

To ensure resilience, each request should follow the same [statechart](https://statecharts.github.io/), as shown below. When all requests, regardless of their source, follow the same pattern, the API client and/or UI can handle them. Each request starts in _idle_. A request can either start or be scheduled. Both from _idle_ and _scheduled_ the request can start. It now moves into the _loading_ state. From this state, four events can happen: success, abort, error, or start. In the latter's case, the previous request is aborted, and a new request is started.

/images/gateway-statechart.png

## Module containers

Modules represent the concept of [domain driven development](https://martinfowler.com/bliki/BoundedContext.html). They implement the [flux pattern](https://facebook.github.io/flux/docs/in-depth-overview/) to arrange business-related logic, state, and UI components. It includes several _components_, as visualized below. Each module has **UI components** and **actions**. They represent the view and the logic of a (business-related) module. Both can interact with the application core. UI components can read from the core, while actions can invoke events in the core and wait for a response.

/images/c4-architecture-level-3-module.png

A module can also have a store. It acts the same as the application store. The store in a module is often used for modeling business logic. It is recommended to follow the described [UI performance principles](#ui-performance-principles).

> **NOTE**: the store can be implemented in the same way as the store in the core container, or use features from a framework (e.g. React Context).

UI components, actions, and a store are common for most modules. But some modules need other _components_.

- A gateway or section module requires a **router**. It determines which **page** or module the user can interact with.
- A route always associates with a module or a **page**. A page is a specialized component.

### Types of modules

There are three different modules identified in this reference architecture. Nesting of modules is possible, regardless of their types.

- A **gateway** module functions as a wrapper around other modules, based on routing. It provides logic and state towards the nested modules (e.g. the main application is as a gateway module).
- Many applications have many pages related to each other. **Section** modules combine these pages and their business logic into a module (e.g. CRUD pages).
- A **block** module is not related to a specific route but can be used anywhere in the application (e.g. shopping cart).

> **NOTE**: these module types are not exclusive. a _gateway_ can also be a _section_, and a _section_ can also be a _block_.

### User interface components

User interface (UI) components are the most important parts of the application. It requires the most development time. It is where the user sees and interacts with the application. There are three different component types.

- **Layout** components are used for positioning of content (e.g. a Stack component) and are (generally) without styling by default (e.g. no background color). As no business logic is present in these components, they live outside of the modules (e.g. inside a design system).
- **Interaction** components are generic components that allow the user to interact with the application (buttons, links, form elements, etc.). Similar to layout components they are without styling by default, exist outside of the modules (e.g. inside a design system).
- **Content** components hold the user interface around the business logic. These components live within the modules and use layout components. They comprise out of five different elements that interact with each other.

/images/c4-architecture-level-4-component.png

The API is how a component interacts with its parent, another UI component. The parent component can provide values, configuration, and callbacks through the API. The values and configuration are, combined with the component state, used to render the UI.

A user interacts with the UI. This interaction invokes an action. A component can use an action from the module or define the action itself. The action can update the component state or invoke a callback received through the API. The observer of a component listens to the values from the API and the state for changes. When a change happens, it invokes a re-render of the UI and invokes an action.

> **NOTE**: modern UI frameworks like React and Vue handle the described observer. React handles re-renders of the UI, while the life-cycles methods (e.g. `useEffect`) handle invoking actions.

## UI performance principles

Users expect modern web applications to be high performing. Several principles are facilitated by the reference architecture to increase the _perceived_ performance. These principles describe the _where_, _when_, and _how_ of UI state management.

- **[Colocation](https://kentcdodds.com/blog/state-colocation-will-make-your-react-app-faster/) (where)**: UI state should live next to the UI code where possible (component, module or application level). State updates will cause fewer re-renders (of parts) of the UI.
- **[Data flow](https://overreacted.io/writing-resilient-components/#principle-1-dont-stop-the-data-flow) (where)**: the state that lives on a higher level should not be put in the state on a lower level. This breaks the data flow of the observer in the [component architecture](#user-interface-components).
- **[Optimistic UI](https://www.smashingmagazine.com/2016/11/true-lies-of-optimistic-user-interfaces/) (when)**: store the expected result of asynchronous or heavy tasks in the state before the actual task is finished. This is detailed in the diagram below.
- **Transactions (how)**: avoid many state mutations at the same time should by combining them in a transaction.
- **[Statecharts](https://statecharts.github.io/) (how)**: UI state should be modeled as a statechart as much as possible, to improve the resilience of the UI.

![](images/optimistic-ui.png)

Next to these principles, **pre-fetching** of data (where possible) results in better performance. When using _gateway_ or _section_ modules, different pages need different data. When entering a page (e.g. detail page), data for other pages can already be pre-fetched and put in the application/module store (e.g. for an overview page). Depending on the entry page and what exists in the store(s), different data is pre-fetched.

## Application governance

Governance in applications is vital for digital enterprises. It refers to the ongoing process of managing, monitoring, and auditing the use of the application. This reference architecture supports several forms of governance.

Role-based access management (RBAC) is the most straight forward auditing implementation. This reference architecture facilitates several possibilities in the UI that can be applied.

- As the _router_ is the main entry point of the application, the initial RBAC implementation needs to happen on this level.
- Each _module router_ can apply the same level of RBAC implementation. This allows for complex, nested access rules based on the URL of the application.
- RBAC can be applied inside a UI component or in parent UI components with _conditional visibility_.
- It is possible to add RBAC rules to the actions in _modules_ and _components_.
- RBAC can be added to the _middleware_ of the _API gateway_.

![](images/rbac.png)

Blocking the rendering of the UI is not enough. When applied, _actions_ in modules and components can implement the RBAC rules. 

But governance is not all about auditing and security. Monitoring ensures more information, i.e. context, is available on defects. One can log all pub/sub, AP, navigation and store events. 