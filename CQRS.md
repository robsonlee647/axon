# Command-Query Responsability Separation (CQRS) 

## Definition
An architectural pattern that separates a system into two different areas:
- Commands: deals with the operations that, when executed, produce changes in the system (write)
- Queries: takes care of the operations that are only requesting information (read only)

## Command Model
- The term `Command Model` refers to a `Domain Model` that is optimized for and contains everything that is needed for executing `Commands`.
- `Command Model` must ensure all stateful components can be properly persisted and loaded

### Example of `Command Model`
In a flight booking system the `Command Model` will likely consist of:
- FlightBooking and other relevant Aggregates
- Flight, Booking, Leg, and other Entities
- Origin, Destination, DepartureDateTime, ArrivalDateTime, and other Value Objects
- BookFlight, CancelBooking, UpdateBooking, … Commands
- FlightBooked, BookingCanceled, BookingUpdated, and other Events

## Query Model
The term `Query Model` refers to a `Domain Model` that is optimized for and contains everything needed for processing `Queries`

Consists of two sets of components
- The information storage and retrieval components known as `Projections`
- Query interpretation and processing components

### Example of `Query Model`
In a flight booking system, the Query Model will likely consist of:
- FlightProjection and other relevant Projections
- Infrastructure components that interact with appropriate storage (RDBS, noSQL DB, file system, etc.)
- Domain components with handleBookingDetailsQuery, handleFlightInformationQuery, handleMatchingFlightsQuery, and other functions
- BookingDetailsQuery, FlightInformationQuery, MatchingFlightsQuery, and other Queries

## Synchronizing the two models
- Use the same storage infrastructure (same database, same file storage, etc.)
- Synchronize on the infrastructure level (stored procedures, triggers, etc.)
- Synchronize via `Events`

## Benefits of CQRS
- Simpler models
- Data access flexibility
- Independent evolution
- Independent performance optimizations
- Independent scalability

## Challenges with CQRS
- Consistency
- Complex Processes
- Mixed model problem (a.k.a. behavior vs information contest)
- Tight coupling the two models
    - Dedicated module
    - Projections in the Command Model
    - Domain Services

## Message
Unit of communication between components. It may carry a `Command`, `Query` or `Event`
- Message granularity: several Messages as a single Message
- Data vs. metadata: distinguishing the data (message’s payload) from the data about the data (message’s metadata)
