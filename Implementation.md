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