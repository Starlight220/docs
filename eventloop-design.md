# Event API Design Proposal
This is a design document for a proposed expansion of 2020+ Command-Based's `Trigger` API into a full-fledged event-based API.


## Goals
To provide an elegant, functional, expressive, non-verbose API for event-based programming in the FRC/WPILib environment, while offering the following features (listed in arbitrary order):
- Polling bindings at differing frequencies.
- Swapping/Disabling bindings.
- Seamless integration with the command-based paradigm *(high priority!)*.


## Core API
The core API would consist of two main classes: `EventLoop` and `BooleanEvent`. 

### `EventLoop`
Contains the event-action bindings, providing a `void bind(BooleanSupplier, Runnable)` method to register new bindings and a `void poll()` method to poll the bindings: Polling bindings is the user's responsibility, letting them poll at whatever frequency they want—similar to the `calculate()` method of various controllers. A `void clear()` method is provided in case a user wants to clear all bindings—though clearing and rebinding should not be recommended.


### `BooleanEvent`
Represents a digital signal, similar in concept to command-based's `Trigger` class. Each event object is associated with a `EventLoop` instance from construction-time. Provides a `void whenTrue(Runnable)` method to bind actions to be executed each polling of the associated `EventLoop` instance if the represented digital signal / condition evaluates to `true`. A `boolean get()` method is provided for querying the signal; classes overriding `BooleanEvent` with custom functionality should either override this method or call `super()` with an argument fulfilling the wanted functionality.

Named `BooleanEvent` to leave room for other types of `*Event` classes in the future.

#### Event Composition
In all composition decorators, the result event inherits the `EventLoop` instance association from the event being composed (in unary compositions) or the event in the `lhs`/`this` position (in binary compositions).

- `negate()`/`operator!()`: Creates a new event that is active when this event is inactive, i.e. that acts as the negation of this event.

- `debounce(seconds_t, DebounceType)`: Creates a new debounced event from this event - it will become active when this event has been active for longer than the specified period.

- `rising()`: Creates a new event that becomes `true` when this event newly changes to `true`.

- `falling()`: Creates a new event that becomes `true` when this event newly changes to `false`.

- `or(BooleanEvent)`/`operator||(BooleanEvent)`: Composes this event with another event, returning a new event that is active when either event is active.

- `and(BooleanEvent)`/`operator&&(BooleanEvent)`: Composes this event with another event, returning a new event that is active when both event are active.

- `withLoop(EventLoop)`: Creates a new event equivalent to this one, with the sole difference of being associated with the given `EventLoop` instance. 
  - This is the only decorator that does not copy the `EventLoop` association.


## Design Decisions and Considered Alternatives
### Loop-Event Association
1. ***Associating at construction-time***
2. Associating at action-bind-time
3. Global Association

With option (1), semantics of composing events associated with different `EventLoop` instances are necessarily asymetric (the loop association of `lhs`/`this` is inherited).
Option (2) simplifies association of composition semantics, but adds verbosity to the binding call.
Option (3) is the current situation with `Trigger` with `CommandScheduler`, and solves the disadvantages of both (1) and (2), but features such as swapping bindings or polling bindings at different frequencies are impossible—defeating the entire point.

Therefore, option (1) was chosen.


## Open Questions
- Multiple bindings and/or `get()` calls can mess up functionality, especially in cases where calling `get()` mutates the object (for example: debounced and falling/rising events). Should events support multiple bindings? If not, how would that be enforced? (C++ might be able to do this with `unique_ptr`). `get()` being `const` (C++) might help this problem. Maybe not provide the `get()` method at all? If `get()` is not `const`, how can `BooleanEvent` objects be sent to telemetry without mutating themselves?


## Future Steps:
- An `EventRobot` base class.
