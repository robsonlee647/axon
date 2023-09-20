# `Commands`
How to implement a Command - an expression of intent to trigger an action in the domain.

With Axon Framework, a Command is represented by a POJO (Plain Old Java Object) that contains:
- The data of the Command
- Minimal routing information.

a `Command` needs to provide:
- `Aggregate`: A component that is instantiated by the framework every time a Command is received
- `Command Handling Component`: A component instantiated by you and registered with the framework during initial configuration



## A `Command` targeting a specific `Aggregate`
If you want to define a `Command` that will be delivered to a specific Aggregate, you need to tell the framework which aggregate instance to load.
```
public class ScheduleFlightCommand {

  @TargetAggregateIdentifier
  private final String id;
  // other state


  public ScheduleFlightCommand(String id /*, other state */) {
    this.id = id;
    // setting state
  }


  // getters, equals, hashCode, toString
}
``` 

Alternative with Lombok:
```
@Value
public class ScheduleFlightCommand {

  @TargetAggregateIdentifier
  private final String id;
  // other state
}
```

## A `Command` targeting specific `Command Handling Component`
a `Command` that will be delivered to a specific `Command Handling Component` (something other than an `aggregate`)
```
public class ScheduleFlightCommand {

  @RoutingKey
  private final String id;
  // other state


  public ScheduleFlightCommand(String id /*, other state */) {
    this.id = id;
    // setting state
  }


  // getters, equals, hashCode, toString
}
```
By annotating the `id` field with `@RoutingKey` you can be sure that all instances of ScheduleFlightCommand that have the same value in the `id` filed will be routed to the same `Command Handling Component`.

Alternative with Lombok:
```
@Value
public class ScheduleFlightCommand {

  @RoutingKey
  private final String id;
  // other state
}
```

# `Command Bus`
How to send a Command using the Command Bus

The process of dispatching a `Command` consists of the following steps:
- Obtain a `Command Bus` instance.
- Convert your `Command` to a `Command Message`.
- Dispatch the `Command Message` using the `Command Bus` instance.

## Getting ready to dispatch a `Command Message`

```
// Obtain a `CommandBus` instance
CommandBus commandBus = configuration.commandBus();

// convert your POJO command to CommandMessage
CommandMessage<ScheduleFlightCommand> commandMessage =
       GenericCommandMessage.asCommandMessage(new ScheduleFlightCommand("AM1"));
```

## Dispatch a `Command Message` without expecting a result
You will get a NoHandlerForCommandException exception if the destination is unreachable, but you will never know if the handler executed successfully or not.
```
// dispatch and forget
commandBus.dispatch(commandMessage);
```

## Dispatch a `Command Message` and react according to the result
This approach gives you the opportunity to properly handle failures (ignore, recover, retry, …).
```
// dispatch with callback function
commandBus.dispatch(commandMessage, (message, result) -> {
    if (!result.isExceptional()) {
       // Successful
    } else {
       // Exceptional
    }
});
```

# `Command Gateway`
Even easier way to interact with the Command Bus via the CommandGateway class:
- Automatically converting a POJOs representing Commands to Command Messages
- Returning CompletableFuture, which you can chain, group, and/or process later

## Getting ready to dispatch a `Command`

```
// Obtain a `CommandGateway` instance
CommandGateway commandGateway = configuration.commandGateway();
```

## Non-blocking
You can use the `send` method. You will get a `CompletableFuture` to check the result later.
```
CompletableFuture<Object> resultAsync =
   commandGateway.send(new ScheduleFlightCommand("AM1"));

resultAsync.thenRun(() -> /* Successful */)
           .thenApply(result -> /* Successful */)
           .exceptionally(exception -> /* Exceptional */);
```

Or you can provide an inline function:
```
commandGateway.send(new ScheduleFlightCommand("AM1"), (message, result) -> {
    if (!result.isExceptional()) {
       // Successful
    } else {
       // Exceptional
    }
});
```


## Blocking
If you want to wait for the result before proceeding, then you should use the sendAndWait method:
```
try {
   Object resultSync = commandGateway.sendAndWait(new ScheduleFlightCommand("AM1"));
   // Successful
} catch (Exception e) {
   // Exceptional
}
```

If you do not want to wait indefinitely, you can provide a timeout:
```
try {
   Object resultSync = commandGateway.sendAndWait(
      new ScheduleFlightCommand("AM1"), 500, TimeUnit.MILLISECONDS
   );
   // Successful
} catch (Exception e) {
   // Exceptional
}
```

# `Command Handlers`
Functions that accept an incoming Command and perform a respective action

```
@CommandHandler
public void handle(ScheduleFlightCommand command) {
   // perform command handling task …
}
```

## `Command Handling Components` 
A single `Command Handling Component`:
- contains one or more `Command Handlers`
- can handle multiple `Commands` and thus contain multiple `Command Handlers`

```
public class CommandHandlingComponent {

   @CommandHandler
   public void handle(ScheduleFlightCommand command) {
       // perform command handling task
   }

   @CommandHandler
   public void handle(CancelFlightCommand command) {
       // perform command handling task
   }
}
```

## Multiple `Command Handlers` with the same signature aren’t allowed

```
/*
 * THIS WILL NOT WORK
 * BECAUSE IS USING THE SAME COMMAND
 */
public class CommandHandlingComponent {

   @CommandHandler
   public void handle(ScheduleFlightCommand command) {
       // perform command handling task
   }

   @CommandHandler
   public void handleFlightSchedule(ScheduleFlightCommand command) {
       // perform command handling task
   }
}
```

## Additional parameters to `Command Handlers`
```
@CommandHandler
public void handle(
    ScheduleFlightCommand command,                          // the message payload
    Message<ScheduleFlightCommand> message,                 // the entire message
    CommandMessage<ScheduleFlightCommand> commandMessage,   // entire command message (a subtype of message)
    MetaData metadata,                                      // the meta data of the message
    @MetaDataValue("name") String name,                     // the value of the "name" meta data key
    @MessageIdentifier String id,                           // the unique identifier of the message
    UnitOfWork<?> unitOfWork,                               // the current unit of work
    @Autowired MySpringBean mySpringBean,                   // instance of MySpringBean
    @Autowired AnotherSpringBean anotherSpringBean,         // instance of AnotherSpringBean
    AnotherMessageHandler another,                          // instance of AnotherMessageHandler registered with the framework
    YetAnotherMessageHandler another                        // instance of YetAnotherMessageHandler registered with the framework
) {
  // perform command handling task …
}
```

# Registering `Command Handling Components`
You need to register it with Axon Framework so it knows where to send each `Command`
## Registration via Spring
```
@Component
public class CommandHandlingComponent {
   // command handling methods
}
```
or
```
@Bean
public class CommandHandlingComponent {
   // command handling methods
}
```

# `Aggregates`
Unlike `Command Handling Components`, which you need for instantiating and registering with the Axon Framework manually. `Aggregates` are instantiated by the framework on demand and their state is restored from the data stored in a `Repository`
```
import static org.axonframework.modelling.command.AggregateLifecycle.apply;

public class Flight {

   private String aggregateId;
   // other state ...

   public void scheduleFlight(String flightId) {
       apply(new FlightScheduledEvent(flightId /*, other state */));
   }
}
```

## Calling the `Aggregate` from the `Command Handler`
`Command Handlers` usually delegate the task execution to the respective `Aggregates`

```
import org.axonframework.modelling.command.Repository;

public class FlightCommandHandler {

   private final Repository<Flight> repository;

   public FlightCommandHandler(Repository<Flight> repository) {
        this.repository = repository;
   }

   @CommandHandler
   public void handle(ScheduleFlightCommand command) {
       repository.load(command.getFlightId())
                 .execute(flight -> flight.scheduleFlight(command.getFlightId()));
   }
}
```

## Combining your `Aggregate` and `Command Handler` into a single class
```
import static org.axonframework.modelling.command.AggregateLifecycle.apply;

public class Flight {
   // Automatically load the Aggregate and find the Repository  
   @AggregateIdentifier
   private String aggregateId;
   // other state ...

   @CommandHandler
   public void handle(ScheduleFlightCommand command) {
       apply(new FlightScheduledEvent(command.getFlightId(), ...));
   }
}
```

## Registering `Aggregates` via `Spring`
```
import org.axonframework.spring.stereotype.Aggregate;

@Aggregate
public class Flight {
   // state and command handlers...
}
```
Furthermore, you can provide additional configuration using respective annotation fields. For example:
```
import org.axonframework.spring.stereotype.Aggregate;

@Aggregate (
   repository = /* configure repository */,
   cache = /* configure cache */
   // other configuration
)
public class Flight {
   // state and command handlers...
}
```

# `Aggregate Members`
Recall from `DDD` that `Aggregates` consist of one or more `Entities` and `Value Objects`

Axon Framework cannot help you much with `Value Objects`(immutable). Creating and using those remains your responsibility. However, Axon Framework can help you with managing `Entities`. In Axon Framework, `Entities` that are part of an `Aggregate` are called `Aggregate Members`.

## `Single member`
if you needed an Entity inside your Aggregate
```
public class Flight {

   @AggregateMember // SeatMap is an Entity of the Flight
   private SeatMap seatMap;

   // other state and command handlers...
}

public class SeatMap {

   @CommandHandler
   public void handle(ConfigureSeatingCommand command) {
       // update
   }
}

```

## `Multiple members`
if you need more than 1 entity
```
public class Flight {

   @AggregateMember
   private SeatMap seatMap;

   @AggregateMember
   private List<Leg> legs;

   // other state and command handlers...
}
```
also, you can use `map`
```
public class Flight {

   @AggregateMember
   private SeatMap seatMap;

   @AggregateMember
   private Map<String, Leg> legs;

   // other state and command handlers...
}
```
such cases will require you to tell the framework how to identify individual `Entities`:
```
public class Leg {

   @EntityId
   private String legId;

   @CommandHandler
   public void handle(MoveLegDestinationCommand command) {
       // command handling ...
   }

   // other state and command handlers...
}
```
# `Message` routing to `Aggregate Members`

## Routing to a single member
if there is a single `Aggregate Member` instance that can handle a `Command`, that is where the command will be sent to.

## Routing `commands` to one of `multiple members`
The framework will compare MoveLegDestinationCommand.legId to Leg.legId and the instance of the Leg for which both values match will handle the command.
```
public class MoveLegDestinationCommand {

   @TargetAggregateIdentifier
   private final String flightId;
   private final String legId;
   // other state ...

}
```
And the `Entity`:
```
public class Leg {

   @EntityId
   private String legId;

   @CommandHandler
   public void handle(MoveLegDestinationCommand command) {
       // command handling ...
   }

   // other state and command handlers...
}
```
However, if field names in the aggregate and the entity differ, you will have to tell the framework what field to use by using a `routingKey`
```
public class MoveLegDestinationCommand {

   @TargetAggregateIdentifier
   private final String flightId;
   private final String legId;
   // other state ...

}
```
And the `Entity`:
```
public class Leg {

   @EntityId(routingKey = "legId")
   private String id;

   @CommandHandler
   public void handle(MoveLegDestinationCommand command) {
       // command handling ...
   }

   // other state and command handlers...
}
```

# Declaring an `Event`
An `Event` is a POJO (Plain Old Java Object) that contains some data and it should be immutable.
```
public class FlightScheduledEvent {

   private final String flightId;
   // other state

   public FlightScheduledEvent(String flightId /*, other state */) {
       this.flightId = flightId;
   }

   // getters, equals, hashCode, toString
}
```
Alternative with Lombok:
```
@Value
public class FlightScheduledEvent {

  private final String flightId;
  // other state
}
```

## Routing information
Unlike Commands, Events do not contain routing information. This is due to one of the most significant differences between those Message types:
- A Command must be delivered to **ONE** specific handler.
- An Event must be delivered to **ALL** handlers interested in it.

# Applying `Events` in `Aggregates`
The message processing lifecycle from an `Aggregate`'s point of view looks like this:

- Load my own state
- Process the incoming Command
- Tell everyone interested what just happened (by applying an Event)
- Update my own state if needed
   - `State Stored Aggregates` (those where you store only the latest state), the order of the execution of last two steps is irrelevant.
   - `Event-Sourced Aggregates` (where you do not store state but instead store each of the state changing events), the order is significant. Thus, one **MUST** update the state from the event and not send an event about the new state.

```
import static org.axonframework.modelling.command.AggregateLifecycle.apply;

public class Flight {

   @CommandHandler
   public void handle(ScheduleFlightCommand command) {

      // validate input, execute business logic, generate output, ...

      apply(new FlightScheduledEvent(command.getFlightId() /*, other state */);
   }

   // methods that update aggregate's state from FlightScheduledEvent
}
```
# Publishing generic `Events`
Generic `Events` that can be used by any component to communicate a ‘fact’ to other domain components

The process of publishing an Event consists of the following steps:
- Obtain an `Event Bus` instance
- Convert your `Event` to an `Event Message`
- Publish the `Event Message` using the `Event Bus` instance

## `EventBus`
```
EventBus eventBus = configuration.eventBus();

FlightScheduledEvent event = new FlightScheduledEvent("AM1");
EventMessage<FlightScheduledEvent> eventMessage = GenericEventMessage.asEventMessage(event);

eventBus.publish(eventMessage);
```

## `EventGateway`
`EventGateway` improves your development experience by automatically converting the POJOs representing `Events` to `Event Messages`
```
EventGateway eventGateway = configuration.eventGateway();

FlightScheduledEvent event = new FlightScheduledEvent("AM1");

eventGateway.publish(event);
```

# `Event Handlers`
Functions that handle published `Events`.

## Declaring methods as `Event Handlers`
```
@EventHandler // use this annotation
public void on(FlightScheduledEvent event) {
   // perform event handling task …
}
```

## `Event Handling Components`
A component that contains `Event Handlers` is called an `Event Handling Component`
```
public class EventHandlingComponent { // may handle multiples EventHandlers

   @EventHandler
   public void on(FlightScheduledEvent event) {
       // Perform event handling task…
   }

   @EventHandler 
   public void on(FlightCanceledEvent event) {
       // …
   }
}
```

## Additional parameters to `Event Handlers`
```
@EventHandler
public void handle(
    FlightScheduledEvent event,                             // the message payload
    Message<FlightScheduledEvent> message,                  // the entire message
    EventMessage<FlightScheduledEvent> eventMessage,        // entire event message (a subtype of message)
    @Timestamp Instant publishedAt,                         // when the event was published
    MetaData metadata,                                      // the meta data of the message
    @MetaDataValue("name") String name,                     // the value of the "name" meta data key
    @MessageIdentifier String id,                           // the unique identifier of the message
    UnitOfWork<?> unitOfWork,                               // the current unit of work
    @Autowired MySpringBean mySpringBean,                   // instance of MySpringBean
    @Autowired AnotherSpringBean anotherSpringBean,         // instance of AnotherSpringBean
    AnotherMessageHandler another,                          // instance of AnotherMessageHandler registered with the framework
    YetAnotherMessageHandler another                        // instance of YetAnotherMessageHandler registered with the framework
) {
  // perform event handling task …
}
```

## Registering `Event Handling Components` via `Spring`
```
@Component
public class EventHandlingComponent {
   // event handling methods
}
```
or `@Bean`:
```
@Bean
public class EventHandlingComponent {
   // event handling methods
}
```

## `Event-Sourced` Aggregates lifecycle
A type of `Aggregates` that maintain their state in a form of series of Events.

There is two types of Aggregates:
- State-Stored Aggregates > ORM (Object–Relational Mapping) systems > state driven (store the current state)
- Event-Sourced Aggregates > Event-Sourcing > behavior driven (store all past)

### Mapping
```
@EventSourcingHandler
public void on(FlightScheduledEvent event) {
   // set aggregate state...
}
@EventSourcingHandler
public void on(ArrivalTimeChangedEvent event) {
   // set aggregate state...
}
```

### Loading
```
Flight flight = new Flight();

List<DomainEvent> pastEvents = // select the events for given flightId in the right sequence

for (DomainEvent event : pastEvents) {
    flight.on(event);
}
```

### Storing
```
apply(new FlightDelayedEvent(flightId /*, other state */));
```

This will: 
- Store an `Event` of type `FlightDelayedEvent` in the configured `Event Store`.
- Call `this.on(flightDelayedEvent)` method.
- Publish an `Event` of type `FlightDelayedEvent` to all registered `Event Handlers`.

## Decision-making vs state changes
Decision-making happens while handling the Command that the current Aggregate was created to handle. For example:
```
@CommandHandler
public void handle(RegisterArrivalTimeCommand command) {
      if (currentArrivalTime.isBefore(command.getNewArrivalTime())) {
         apply(new FlightDelayedEvent(flightId, command.getNewArrivalTime()));
      } else {
         apply(new ArrivalTimeChangedEvent(flightId, command.getNewArrivalTime()));
      }
}
```

### State changes
All state changes should be done while handling a domain `Event`
```
@EventSourcingHandler
protected void on(ArrivalTimeChangedEvent event) {
      this.currentArrivalTime = event.getNewArrivalTime();
}
```
It will be executed:
- Once for each past `ArrivalTimeChangedEvent` while the `Aggregate` is loaded.
- Every time the `ArrivalTimeChangedEvent` is applied during the life cycle of the `Aggregate` via `apply(new ArrivalTimeChangedEvent(flightId /*, other state */));` method

## Registering `Aggregates`
Since `Aggregates` are instantiated by the framework on demand, you will register a `Java .class` type and not an actual instance of the class.

### Via `Spring`
```
import org.axonframework.spring.stereotype.Aggregate;

@Aggregate
public class Flight {
   // state and command handlers...
}
```
Furthermore, you can provide additional configuration using respective annotation fields
```
import org.axonframework.spring.stereotype.Aggregate;

@Aggregate (
   repository = /* configure repository */,
   cache = /* configure cache */
   // other configuration
)
public class Flight {
   // state and command handlers...
}
```

## Result validation
There is no single then method. Instead, there are plenty of methods designed to assert that the `Aggregate` behaves as expected

### Happy path
- expectEvents(/* ... */) asserts the given set of Events has been published.
- expectNoEvents() asserts no Events have been published.
- expectSuccessfulHandlerExecution() asserts there were no errors.
- expectResultMessagePayload(expectedPayload) asserts the Command Handler returned the expected object.
-  expectResultMessage(GenericCommandResultMessage.asCommandResultMessage(/* ... */)) asserts the Command Handler returned the Message we expect.
- …

### Unhappy path
- expectException(Exception.class) asserts that the given exception occurs during `Command Handler` execution.
- expectExceptionMessage("This is unacceptable") asserts that an exception containing specific message occurs during -  `Command Handler` execution.
- …

## Result validation with `matchers`
A more advanced way to assert desired behavior than simple object comparison
```
FixtureConfiguration<Flight> fixture = new AggregateTestFixture<>(Flight.class);

@Test
void test() {
    fixture.given(/* historical events */)
        .when(new RegisterArrivalTimeCommand(flightId, newArrivalTime))
        .expectEventsMatching(
            Matchers.exactSequenceOf(
                Matchers.messageWithPayload(
                    Matchers.matches(payload -> {
                        // compare payload with expectations
                    })
                )
            )
        );
}
```

### Methods that support `matchers` 
- expectEventsMatching
- expectException
- expectResultMessageMatching
- expectResultMessagePayloadMatching

### Available `matchers`
- Matchers.andNoMore() - matches against null or void.
- Matchers.nothing() - matches against null or void.
- Matchers.noCommands() - matches an empty List of Commands.
- Matchers.noEvents() - matches an empty List of Events.

- Matchers.equalTo(expected) - matches against each Event
- Matchers.equalTo(expected, filter) - matches against each Event

- Matchers.sequenceOf(matchers) - matches a series of Events
- Matchers.exactSequenceOf(matchers) - matches an exact List of Events
- Matchers.listWithAllOf(matchers) - matches a List where all the given matchers must match
- Matchers.listWithAnyOf(matchers) - matches a List where at least one matcher matches
- Matchers.messageWithPayload(matcher) - matches a payload of a Message
- Matchers.payloadsMatching(matcher) - matches the payloads of a List of Messages

# Declaring and sending queries
You learned how to implement a `Query` - a request for information or state.

With `Axon Framework`, a `Query` is a `POJO` (Plain Old Java Object) that contains the data of the `Query`,
when a Query is dispatched, the expected response type must be specified

Based on the `Query` type and the `response` types, the framework will automatically find a respective component that can handle it.

### Declaring a `Query` without `Lombok`
```
public class ScheduleForBookingQuery {

   private final String bookingId;
   // other state

   public ScheduleForBookingQuery(String bookingId) {
       this.bookingId = bookingId;
       // setting state
   }

   // getters, equals, hashCode, toString
}
```

### Alternative with `Lombok`
```
@Value
public class ScheduleForBookingQuery {

  private final String bookingId;
  // other state
}
```

## Direct `Query` (a.k.a `Point-to-Point`)
Queries can be divided into three types:
- Direct Query
- Scatter-gather Query
- Subscription Query

### Direct Query
This is the most straightforward, when you intend to query a single component, receive a response, and do not expect to be updated when changes happen in the future

#### Dispatching `Direct Queries `via `QueryBus`
```
// Obtain a `QueryBus` instance
QueryBus queryBus = configuration.queryBus();

ScheduleForBookingQuery query = new ScheduleForBookingQuery("AM1");
```
Before dispatching a `Query`, you need a `ResponseType`
```
// expect single FlightSchedule
ResponseType<FlightSchedule> oneSchedule =
        ResponseTypes.instanceOf(FlightSchedule.class);

// expect a collection of FlightSchedule objects
ResponseType<List<FlightSchedule>> manySchedules =
        ResponseTypes.multipleInstancesOf(FlightSchedule.class);

// expect optional FlightSchedule
ResponseType<Optional<FlightSchedule>> optionalSchedule =
        ResponseTypes.optionalInstanceOf(FlightSchedule.class);

// That represents the object type of the Response Message
```

With that in place, you can build a `Query Message` and dispatch it.
```
// convert POJO query to QueryMessage
QueryMessage<ScheduleForBookingQuery, FlightSchedule> queryMessage =
        new GenericQueryMessage<>(query, oneSchedule);

CompletableFuture<QueryResponseMessage<FlightSchedule>> result =
        queryBus.query(queryMessage);
```
The result is `CompletableFuture` which contains a `Response Message`. That allows you to operate in both the blocking and non-blocking fashions. If there is an error, the `Response Message` will be marked as exceptional. 
```
queryBus.query(queryMessage)
        .thenApply(responseMessage -> {
            if (!responseMessage.isExceptional()) {
                // Successful
            } else {
                // Exceptional
            }
        });
```

#### Dispatching direct queries via `QueryGateway`
`QueryGateway` is there to simplify the process of interacting with `QueryBus`
```
// instantiating
QueryGateway queryGateway = configuration.queryGateway();
ScheduleForBookingQuery query = new ScheduleForBookingQuery("AM1");
```
Sending a `Query` that specifies the expected result type
```
// expect single FlightSchedule
CompletableFuture<FlightSchedule> oneSchedule =
    queryGateway.query(query, FlightSchedule.class);

// expect a collection of FlightSchedule objects
CompletableFuture<List<FlightSchedule>> manySchedules =
    queryGateway.query(query, ResponseTypes.multipleInstancesOf(FlightSchedule.class));

// expect optional FlightSchedule
CompletableFuture<Optional<FlightSchedule>> optionalSchedule =
    queryGateway.query(query, ResponseTypes.optionalInstanceOf(FlightSchedule.class));
```

An alternative approach is to explicitly specify the name of the `Query`
```
// get single FlightSchedule
CompletableFuture<FlightSchedule> oneSchedule =
    queryGateway.query("flightScheduleQuery", query, FlightSchedule.class);

// get number of scheduled flights
CompletableFuture<Integer> flightsCount =
    queryGateway.query("number_of_scheduled_flights", null, Integer.class);
```

The actual `Query` can be `null`. Another way to verify is:
```
// example of a non-blocking operation that also accounts for errors.
queryGateway.query(query, FlightSchedule.class)
            .thenApply(result -> /* Successful */)
            .thenRun(() -> /* Successful */)
            .exceptionally(exception -> /* Exceptional */);
```

### Scatter-Gather Query
`Axon Framework` supports and it allows you to dispatch a request to multiple handlers and combine the results.

#### Dispatching `Scatter-Gather Queries` via `QueryBus`
Let's construct a Query Message that expects a single schedule as a result. Of course you could request multiple schedules or an optional one instead by providing respective `ResponseType`.
```
// Obtain a `QueryBus` instance
QueryBus queryBus = configuration.queryBus();

ScheduleForBookingQuery query = new ScheduleForBookingQuery("AM1");

// expect single FlightSchedule
ResponseType<FlightSchedule> oneSchedule =
        ResponseTypes.instanceOf(FlightSchedule.class);

// convert POJO query to QueryMessage
QueryMessage<ScheduleForBookingQuery, FlightSchedule> queryMessage =
        new GenericQueryMessage<>(query, oneSchedule);
```

The main difference from `Direct Queries` is how you dispatch a `Scatter-Gather Query`. You need to use the `scatterGather` method and provide a timeout.
```
Stream<QueryResponseMessage<FlightSchedule>> result =
        queryBus.scatterGather(queryMessage, 500, TimeUnit.MILLISECONDS);
```

The result is a `Stream` of all `Response Messages` received by relevant query handlers. The keyword here is ALL messages. That includes the exceptional ones. Therefore you will have to filter out the exceptional ones yourself.
```
queryBus.scatterGather(queryMessage, 500, TimeUnit.MILLISECONDS)
        // perform any stream operation, e.g.
        .filter(responseMessage -> !responseMessage.isExceptional())
        .collect(/* collect operation */);
```

#### Dispatching `Scatter-Gather Queries` via `QueryGateway`
It contains the equivalent scatterGather wrapper methods
```
QueryGateway queryGateway = configuration.queryGateway();
ScheduleForBookingQuery query = new ScheduleForBookingQuery("AM1");

// expect single FlightSchedule from each handler
CompletableFuture<FlightSchedule> oneSchedule =
    queryGateway.scatterGather(query, FlightSchedule.class, 500, TimeUnit.MILLISECONDS);

// expect a collection of FlightSchedule objects from each handler
CompletableFuture<List<FlightSchedule>> manySchedules =
    queryGateway.scatterGather(query, ResponseTypes.multipleInstancesOf(FlightSchedule.class), 500, TimeUnit.MILLISECONDS);

// expect optional FlightSchedule from each handler
CompletableFuture<Optional<FlightSchedule>> optionalSchedule =
    queryGateway.scatterGather(query, ResponseTypes.optionalInstanceOf(FlightSchedule.class), 500, TimeUnit.MILLISECONDS);
```

As with Direct Queries, there is also the alternative approach where you specify the query name
```
// get single FlightSchedule
CompletableFuture<FlightSchedule> oneSchedule =
    queryGateway.scatterGather("flightScheduleQuery", query, FlightSchedule.class, 500, TimeUnit.MILLISECONDS);

// get number of scheduled flights
CompletableFuture<Integer> flightsCount =
    queryGateway.scatterGather("number_of_scheduled_flights", null, Integer.class, 500, TimeUnit.MILLISECONDS);
```

The result here is again a `CompletableFuture`, but it contains the actual good response extracted from the `Response Message`
```
queryGateway.scatterGather(query, responseType, 500, TimeUnit.MILLISECONDS)
            // perform any stream operation, e.g.
            .map(result -> /* Successful */)
            .collect(/* collect operation */);
```

#### Important note about `ResponseType` in `Scatter-Gather Queries`
The `ResponseType` in `Scatter-Gather Queries` indicates the type of the elements returned by each query handler. In other words, the type of elements contained in the Stream. It does not indicate the total number of responses you will get.

For example, if there are five components that can handle ScheduleForBookingQuery and all of them respond successfully and on time, the resulting Stream will contain:
- five `A` objects when ResponseTypes.instanceOf(A.class)
- five `List<A>` objects when ResponseTypes.multipleInstancesOf(A.class)
- five `Optional<A>` objects when ResponseTypes.optionalInstanceOf(A.class)

### `Subscription Query`
Supported by `Axon Framework`, and it not only allows you to get immediate results but also to subscribe to future changes of the results.

#### Dispatching `Subscription Queries` via `QueryBus`
The first difference while dispatching them is that you can provide different Response Message types for the initial result and for the updates

```
ResponseType<FlightSchedule> initialResponseType =
    ResponseTypes.instanceOf(FlightSchedule.class);

ResponseType<FlightScheduleUpdate> updateResponseType =
    ResponseTypes.instanceOf(FlightScheduleUpdate.class);
```

The second difference is how you constructed the `Query Message`. There is a dedicated type for `Subscription Queries` in `Axon Framework` called `SubscriptionQueryMessage`. To construct an instance you need to provide both the expected initial response type and the update response type. 
```
SubscriptionQueryMessage<ScheduleForBookingQuery, FlightSchedule, FlightScheduleUpdate> queryMessage =
    new GenericSubscriptionQueryMessage<>(query, initialResponseType, updateResponseType);
```

The final difference is the use of a dedicated method for subscription queries called `subscriptionQuery`.
```
SubscriptionQueryResult<QueryResponseMessage<FlightSchedule>, SubscriptionQueryUpdateMessage<FlightScheduleUpdate>>
   result = queryBus.subscriptionQuery(queryMessage);
```

#### Dispatching `Subscription Queries` via `QueryGateway`
QueryGateway contains the equivalent wrapper methods:
```
QueryGateway queryGateway = configuration.queryGateway();

ScheduleForBookingQuery query = new ScheduleForBookingQuery("AM1");

SubscriptionQueryResult<FlightSchedule, FlightSchedule> result =
   queryGateway.subscriptionQuery(query, FlightSchedule.class, FlightSchedule.class);

SubscriptionQueryResult<List<FlightSchedule>, FlightSchedule> initialListResult =
   queryGateway.subscriptionQuery(query,
                                  ResponseTypes.multipleInstancesOf(FlightSchedule.class),
                                  ResponseTypes.instanceOf(FlightSchedule.class));
```

An alternative approach is to explicitly specify the Query name
```
// get single FlightSchedule
SubscriptionQueryResult<FlightSchedule, FlightSchedule> namedResult =
   queryGateway.subscriptionQuery("flightScheduleQuery",
                                  query, FlightSchedule.class, FlightSchedule.class);

// get number of scheduled flights
SubscriptionQueryResult<Integer, Integer> flightsCount =
    queryGateway.subscriptionQuery("number_of_scheduled_flights", null, Integer.class, Integer.class);

// Query can be null!
```

There are also two ways of handling the results:
- Through the Axon Framework’s API
- Through the Project Reactor’s API that the framework uses internally

#### Handling the `responses` from `QueryBus`
Using `Axon Framework’s API` only
```
result.handle(
    // handle the initial response
    initialResponseMessage -> {
        if (!initialResponseMessage.isExceptional()) {
            // Successful
            FlightSchedule flightSchedule = initialResponseMessage.getPayload();
            // ...
        } else {
            // Exceptional
            Throwable error = initialResponseMessage.exceptionResult();
            // ...
        }
    },

    // handle the update response
    updateMessage -> {
        if (!updateMessage.isExceptional()) {
            // Successful
            FlightSchedule flightSchedule = updateMessage.getPayload();
            // ...
        } else {
            // Exceptional
            Throwable error = updateMessage.exceptionResult();
            // ...
        }
    }
);
```

The same can be done with the `Project Reactor’s API`:
```
// handle the initial response
result.initialResult().doOnNext(initialResponseMessage -> {
    if (!initialResponseMessage.isExceptional()) {
        // Successful
        FlightSchedule flightSchedule = initialResponseMessage.getPayload();
        // ...
    } else {
        // Exceptional
        Throwable error = initialResponseMessage.exceptionResult();
        // ...
    }
}).subscribe();

// handle the update response
result.updates().doOnNext(updateMessage -> {
    if (!updateMessage.isExceptional()) {
        // Successful
        FlightSchedule flightSchedule = updateMessage.getPayload();
        // ...
    } else {
        // Exceptional
        Throwable error = updateMessage.exceptionResult();
        // ...
   }
}).subscribe();
```

#### Handling the `responses` from `QueryGateway`
Using `Axon Framework’s API` only:
```
result.handle(
    // handle the initial response
    flightSchedule -> {
        // ...
    },
    // handle the update response
    flightSchedule -> {
        // ...
    }
);
```

The same can be done with the `Project Reactor’s API`:
```
// handle the initial response
result.initialResult().doOnNext(flightSchedule -> {
    // ...
}).subscribe();

// handle the update response
result.updates().doOnNext(flightSchedule -> {
    // ...
}).subscribe();
```

This gives you the opportunity to use the Project Reactor features
```
// handle both initial response the update response(s)
result.initialResult()
    .concatWith(gatewayResult.updates())
    .doOnNext(flightSchedule -> {
        // ...
    }).subscribe();

// it is a way to combine the initial and update object(s) and process them together (assuming they are of the same type).
```

Canceling the subscription
Once subscribed to receive updates, they will be received until one of the following happens:
- The subscription is explicitly canceled.
- The query handler completes the update process (for example, when there are no additional updates possible).
- The application is closed.
```
result.close();
```

## `Query Handlers`
Functions that handle and respond to queries.

### Declaring methods as `Query Handlers`
```
@QueryHandler
public Itinerary handle(FindItineraryQuery query) {
   // retrieve or construct response …
   return /* result */;
}
```
The code above states that the handle method can handle `FindItineraryQuery`. Therefore, when such a `Query` is received, the `Message Bus` will call this method and pass the respective object to it. The `Itinerary` object returned by the method will be the payload of the `Response Message` delivered back to the component dispatching the Query.

### `Query Handling Components`
A component that contains `Query Handlers` is called `Query Handling Component`.
```
public class QueryHandlingComponent {

   @QueryHandler(queryName = "findItinerary")
   public Itinerary handle(FindItineraryQuery query) {
       // retrieve or construct response …
       return /* result */;
   }

   @QueryHandler
   public List<FlightSchedule> handle(AllFlightSchedulesQuery query) {
       // retrieve or construct response …
       return /* result */;
   }
}
```

### `Subscription Query` Updates
It differs from the rest of the query handling (`Direct Queries`, `Scatter-Gather Queries`) because it needs answers to the following questions:

- When do the updates occur?
- How are the updates are sent back?
- Which components need to be updated?

```
@EventHandler
public void on(FlightStatusUpdatedEvent event) {

    // 1. update the data in the projection respectively

    // 2. send updates to subscription queries operating on this data about the change
}
```

#### How the updates are sent back?
There is a dedicated infrastructure component in Axon Framework called QueryUpdateEmitter that is used to deliver all updates
```
private final QueryUpdateEmitter queryUpdateEmitter = configuration.queryUpdateEmitter();
```

#### Completing subscriptions
When you indisputably know that there are no more possible updates (for example, the fight has arrived or has been canceled), you can complete all subscriptions with the complete method:
```
queryUpdateEmitter.complete(
    ChatMessageQuery.class,
    query -> query.getFlightId().equals(event.getFlightId())
);
```

#### Additional parameters to Query Handlers
```
@EventHandler
public void handle(
    FindItineraryQuery query,                               // the message payload
    Message<FindItineraryQuery> message,                    // the entire message
    QueryMessage<FindItineraryQuery> queryMessage,          // entire query message (a subtype of message)
    MetaData metadata,                                      // the meta data of the message
    @MetaDataValue("name") String name,                     // the value of the "name" meta data key
    @MessageIdentifier String id,                           // the unique identifier of the message
    UnitOfWork<?> unitOfWork,                               // the current unit of work
    @Autowired MySpringBean mySpringBean,                   // instance of MySpringBean
    @Autowired AnotherSpringBean anotherSpringBean,         // instance of AnotherSpringBean
    AnotherMessageHandler another,                          // instance of AnotherMessageHandler registered with the framework
    YetAnotherMessageHandler another                        // instance of YetAnotherMessageHandler registered with the framework
) {
  // perform query handling task …
}
```

### Registering `Query Handling Components`
You may register via `Configuration API` or via `Spring`

#### Registration via Spring
Annotating them with @Component
```
@Component
public class QueryHandlingComponent {
   // query handling methods
}
```
or with @Bean
```
@Bean
public class QueryHandlingComponent {
   // query handling methods
}
```

## Projections
Components responsible for maintaining (subsets of) data in a form and place that is optimized for given consumer.

### Building and updating Projections
Projections are updated when a relevant Events occur.
```
public class PassengerFlightTimeProjection {

    DataStorage dataStorage = /* optimal storage for the use case */

    @EventHandler
    public void on(FlightDelayedEvent event) {
        FlightSchedule flightSchedule = dataStorage.loadFlightSchedule(event.getFlightId());
        flightSchedule.updateArrivalTime(event.getNewArrivalTime());
        dataStorage.store(flightSchedule);
    }

    // other event handlers you might need
}
```

### Querying Projections
Projections provide ways for consumers to request the desired data. This is why they are Query Handling Components and have @QueryHandler annotated method(s).
```
public class PassengerFlightTimeProjection {

    DataStorage dataStorage = /* optimal storage for the use case */

    @EventHandler
    public void on(FlightDelayedEvent event) {
        FlightSchedule flightSchedule = dataStorage.loadFlightSchedule(event.getFlightId());
        flightSchedule.updateArrivalTime(event.getNewArrivalTime());
        dataStorage.store(flightSchedule);
    }

    // other event handlers you might need

    @QueryHandler
    public FlightSchedule handle(ScheduleForBookingQuery query) {
        return dataStorage.loadFlightSchedule(query.getBookingId());
    }

    // other query handlers you might need
}
```

### Registering Projections
you can do that via an Axon Framework API:
```
PassengerFlightTimeProjection projection = new PassengerFlightTimeProjection();
Configurer configurer =
   DefaultConfigurer.defaultConfiguration()
                    // any other configuration
                    .registerMessageHandler(config -> projection);
```
you can register them “the Spring way” by annotating them with @Component:
```
@Component
public class PassengerFlightTimeProjection {
   // methods
}
```
Or @Bean:
```
@Bean
public class PassengerFlightTimeProjection {
   // methods
}
```