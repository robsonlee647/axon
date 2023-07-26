# Event Sourcing
It is about using the past Events to make new decisions for future Events.

- Event Sourcing is typically used with CQRS and DDD because it fits naturally into the Command Model where all state changes are made.
- Events the single source of truth

Not the same as Event Streaming

## Event Streaming
- Gets Events from one component to another
- Usually, it consists of temporary storage for the events and a “read forward” procedure. 

## Benefits of Event Sourcing
- It simplifies the modeling process (the domain events you identify dictate the rest).
- It reduces development work (no need to implement state storages).
- It provides greater fault-tolerance because it is possible to recover the state of the system at any point.
- It collects extensive data that you can use for analysis, predictions, etc.

## Event Store
- It refers to an infrastructure component that stores Events
- Should efficiently read all events only by filtering the unsuitable events and without executing a full scan.
- Not every mechanism capable of storing events is an event store.

## Challenges
- Performance challenges
    - Ever-growing data set
    > the application is recorded as `Events`. They are now the only source of truth. The solution is to use a built-for-purpose `Event Store`
    - Efficient data retrieval
        - If you are not using a built-for-purpose Event Store, you must have very carefully designed indexes in place.
        - If you are designing a distributed application, you should seriously consider using a consistent hashing algorithm.
        - In all cases, when you expect to deal with Aggregates with a long history, you should consider using `Snapshots`
- Fixing immutable data
    - What if the data you have already persisted is wrong? How do you fix a mistake in immutable data? 
    > The proper way to deal with mistakes and incorrect data is by adding compensating Events. Acknowledge the error, execute operations that correct it, and append the resulting Events in the history.
- Inside vs outside Events
    - Inside events
    > These are the events that only make sense in context. 
    - Outside events
    > These are the events that carry information across contexts.
- Dealing with sensitive data
    - Out-of-band storage
    > You store any sensitive information in an external system and not inside the Events
    - Crypto erasure
    > You can encrypt the sensitive data using one or more keys stored in a safe place outside the Events
- Changing the structure of the Events
    - Backward and forward compatibility
    > You can establish a policy never to change specific fields with high importance. Then you make sure to treat everything else as optional. Design your `Event Handlers` not to depend on the optional fields by providing conditional logic or default values.
    > Upcasters
    - `Upcasters` are infrastructure components that operate in the space between the Event Store and the Event Handler. They are responsible for transforming a specific revision of a stored Event (one read from the store or already transformed by another upcaster) to a more up-to-date revision of the same Event before passing it to the Event Handler.
- Dealing with infrastructural complexity
    - Use built-for-purpose platform
    > Axon is one such solution that provides a solid messaging platform, abstractions of the terms defined in the above patterns, multiple infrastructure components, and many more. 
    - DIY infrastructure
    > Following capability map will help you keep track of all the functionalities that your solution should provide on your custom infrastructure
    ![Capability map](https://axoniq.github.io/academy-content/assets/img/capability_map.png)
- Domain modelling and application design processes
    - Event storming
    > The term Event storming refers to a rapid group modeling approach to domain-driven design. It is a workshop-style technique that brings project stakeholders together to explore complex business `Domains`. 
    - Event modeling
    > It focuses on details. Its aim is to deliver a working solution rather than a generic approach to designing any system.