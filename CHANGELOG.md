# CHANGE LOG

## Version 0.4.0

- Added module architecture & domain driven design example
- Added the link between the data access layer of the application store and the mediator in the gateway

## Version 0.3.0

- Added the request statechart, including explanation, in the API gateway section;
- Updated legend to reflect the diagrams a bit better;
- Added component architecture;
- Added module architecture.

## Version 0.2.1

- Updated the diagram and text in the 'API gateway' section. The client does not send an event via the pub/sub, as UI components can update UI state around async code themselves, so the do not need an observer pattern. In addition, a link between the store and the pub/sub is added to complete the section around the middleware.
