# CHANGE LOG

## Version 3.0.0

- Introduced new principles, concepts and patterns (CQS, SoC, MVP);
- Added visualization around the layered architecture and MVP. 
- Removed module-type section;
- Removed application governance;
- Created a new architecture for the modules;
- Updated all text to be in line with the new module architecture and the newly introduced principles. 

## Version 2.0.0

Converted the reference architecture and all its diagrams to the [C4 model](https://c4model.com). This has some breaking changes:

- Combined components from the store and API gateway in a higher level, into to level 3 core container;
- Added "dynamic diagrams" for the store and API gateway;
- Added separate section for the pub/sub;
- Removed some elements/components from the API gateway (the circuit breaker) as these are handled now by the gateway component and store container respectively. In addition, the link between the middleware and the pub/sub is removed, as this is now also handled by the gateway component.

## Version 1.0.0

Initial and complete version of the reference architecture, including:

- High level overview;
- Detailed views of the application core, and on lower level the store and API gateway;
- Module architecture;
- Component architecture;
- Application goverance.
