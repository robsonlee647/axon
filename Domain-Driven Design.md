# Domain-Driven Design (DDD)

## Domain Model
- is the `heart` of an application
- contains only `concepts` used in the client's domain that are used to solve a specific problem
- e.g.: 
    - domain = world globe && model = map

## Entity
- objects that are not fundamentally defined by their attributes
- has an identity
- has a thread of continuity

### Example of `Entity`
A bank account (when modeling bank transfers)
- The balance, limits, and overdraft allowance might change, but it is still the same account.
- Other accounts might have the same balance, limits, and overdraft allowance, but they are different accounts.

Therefore, you should probably model a bank account as an Entity.

## Value  Object
- defined by its attributes
- no conceptual identity
- immutable

### Example of `Value Object`
An address
- An address has a city, street, and number. If you change any of the attribute values, it is no longer the same address.
- Any number of addresses with the same values of their fields are indistinguishable from one another and thus essentially are the same object.

Therefore, you should probably model an address as a Value Object. An address does not have an explicit identifier. You don’t need one, and you should not try to invent one.

## Repository
- is conceptual
- infrastructure component that can store, find, and provide instances of Aggregates, Entities and Value Objects

## Aggregate
- boundaries and rules of direct object relationships

### Aggregate Root
- There must be a single object inside the Aggregate that can interact with the objects outside the Aggregate
- If deleted, all objects inside the Aggregate are also deleted

### Example of `Aggregate`/`Aggredate Root`
One Aggregate represents the passenger, itinerary, and legs. Another Aggregate represents each flight.
- The passenger, itinerary, and legs are one unit with regard to data changes.
- There are well-defined relationships: The itinerary belongs to the passenger; the itinerary consists of legs.
- There are invariants: the origin of the first leg and the destination of the last one never changes.
- The itinerary is what is important to the outside object, so it is the Aggregate Root.
- References to the passenger and individual legs from outside objects do not make sense and are not allowed.
- Legs have references to flights (different Aggregate) that may change without affecting the itinerary.

## Event
- Notification that something relevant has happened inside the domain

## Bounded Context
- set boundaries in terms of team organization, usage within specific parts of the application, physical form
- keep model strictly consistent within bounds

### Example of `Event`
Item added to the shopping cart:
- This event is likely to be only relevant in the `Aggregate` responsible for the given shopping cart.
- Note the past tense.
- You could, in response, execute a “remove an item from shopping cart” operation, which will result in reverting the state and emitting an “Item removed from the shopping cart” compensating event.

An order has been received:
- This event is likely to be only relevant in the `Bounded Context` responsible for managing orders. 
- There may be multiple components that need to react somehow to that fact (check product availability, calculate discount prices, etc.). 

### Example of `Bounded Context`
Consider the “flight” object and the passenger view:

- A passenger most likely cares about the flight number, the check-in time, the gate, and how much luggage is allowed. 

The passenger context of your model should take those specific concerns into account and ignore all the other information that is irrelevant from the passenger’s perspective.

