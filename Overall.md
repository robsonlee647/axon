# Concepts

</br>

## Domain
specific sphere of knowledge influence or activity to which a program is applied.
## Model
system of abstractions that describe a subject in a way that people can know, understand or simulate it
## Domain Model
minimal useful Model of the actual Domain
## Command Model
a Domain Model designed and optimized for executing Commands (state changing operations)
## Query Model
a Domain Model designed and optimized for executing Queries (read-only operations)
## Bounded Context
defines a specific part of the Domain Model in which specific objects have consistent meaning and relevant characteristics
## Domain Service
refers to a piece of business logic implemented independently so that it could be called from different handlers, either from a Command Handler and/or from a Query Handler.
## Location Transparency
indicates that the system components do not need to know where other components are running. Components communicate via Messages delivered by a Message Bus
## Event Storming
combination of events driven and brainstorming
## Event Modeling
focuses on the details and aims more on delivering a working solution in software, rather than being a generic approach to designing any type of system

</br> </br></br>

## Entity
a Domain Model object that is defined by a thread of continuity and identity
## Value Object
a Domain Model object that has no conceptual identity, but is fundamentally defined by the values of its attributes
## Aggregate
group of associated components which are considered as one unit with regard to data changes.
## Event-Sourced Aggregate
an Aggregate that stores all past Events and reconstructs its state from them when needed.
## Command
expression of intent to trigger an action in the domain
## Query
a request for information or state
## Event
a notification that something relevant has happened inside the Domain
## Projection
a component responsible for maintaining (subsets of) data in a form and place that is optimized for given consumer
## Snapshot
stored state of an Aggregate at a particular point in time

</br> </br></br>

## Message
unit of communication between components ensuring Location Transparency
## Message Bus
infrastructure component supporting sending and receiving messages between domain components
## Repository
infrastructure component responsible for storing, finding and loading stateful Domain Model objects
## Event Store
infrastructure component that stores Events
## Upcaster
infrastructure components responsible for transforming Event stored using old structure to more up-to-date structure