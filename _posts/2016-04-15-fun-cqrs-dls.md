
#  A CQRS Behavior DSL

When we started the development of Fun.CQRS one of our goals was to come up with a comprehensive API that we could reuse in different internal projects. The API needed to be minimalistic and only expose the basic operations that all CQRS / ES need. It needed also to be flexible enough to accommodate use cases that we didn't think about in the beginning. 

The choice for **Akka** as a backend was easy. Our team had already experience with it and **Akka Persistence** had almost all the machinery we needed for Event Sourcing. The only missing piece at the time was the **Query** capability. For that we had to develop our own system to query and stream events to our **Projections**.  This was later replaced by the new experimental **Akka Persistence Query** model.

**Akka** gave us **Event Sourcing**, so we need to something to help us modeling our domain in terms of **Commands** and **Events**. And of course, using our favorite Scala style of modeling, i.e: case classes and functions.

Defining the basic operations we needed was easy, the challenge was how to make it work inside **Akka** in a transparent way, not because we were not happy with **Akka**, on the contrary, but because our feeling was that we should abstract out everything that was not directly related with the business domain.

We had come up the following specification:  

* Whatever it happens, everything must be expressed in terms of **Commands** and **Events**.   
* Every single aggregate has at least two phases, whether it is initialized whether it is not.  
Uninitialized aggregates need a **Factory** that takes the role of a constructor. It accepts **Commands**, validates it and emits one or more **Event** (initially we thought that we should generate only one Event at construction type, we changed our mind later on).

The flow is well known. 

 1. A **Command** is sent to an existing **Aggregate** or to its **Factory**
 - **Command** is validated, if valid one or more **Events** will be emitted
 - **Events** are persisted (but this must be transparent to the user)
 - **Events** are applied
  * if at construction time, a new **Aggregate** is instantiated
  * if it's an existing **Aggregate**, events are applied producing a new state
 - **Aggregate** is ready to accept the next **Command**

Although in the majority of the cases we will produce one single Event, we need to support the possibility to produce more than one Event in some situations.
 
 Although in the majority of the cases we will be validating a Command solely against the current state of the aggregate, in some situations we need to run a query or call an external service. 
 
Given the specifications above we came up with the following functions signatures:

```scala
    // when constructing
    Command => Event
    Command => Future[Event] // for asynchronous validation
    Command => Try[Event] // for synchronous validation of commands that may fail
    Event => Aggregate // apply Event and build the aggregate
    
    // when updating existing aggregates
    Aggregate => Command => Event
    Aggregate => Command => List[Event] // when more than one event is emitted    
    Aggregate => Command => Future[Event] // for asynchronous validation
    Aggregate => Command => Future[List[Event]] // for asynchronous validation with many events
    Aggregate => Command => Try[Event] // for synchronous validation of commands that may fail
    Aggregate => Command => Try[List[Event]] // for synchronous validation of commands that may fail
    Aggregate => List[Event] => Aggregate // apply event and update aggregate
```

# Behavior definition

We can define the behavior of an aggregate as the sum of all functions (command handlers and event handlers) that are defined for it. As such it was clear for us that we need a way to declare all possible command handlers and event handlers for a given aggregate and make sure that they were available and applicable depending on the internal state of the aggregate.

## Declarative Behavior
Our first trial was to define them as different PartialFunctions that we could lift to a common type (Future[List[Event]]) and compose them all to build the final behavior.

This led was to the definition of the following method that we could use to pass the PartialFunction and compose the final behavior.

```scala
// non-exhaustive listing 
// when constructing
def acceptsCommand(handler: PartialFunction[Command, Event]))
def acceptsCommandAsync(handler: PartialFunction[Command, Future[Event]]))
def acceptsCommandTry(handler: PartialFunction[Command, Try[Event]]))
def appliesEvent(handler: PartialFunction[Event, Aggregate]))

// when updating
def acceptsCommand(handler: PartialFunction[(Aggregate, Command), Event]))
def acceptsCommandAsync(handler: PartialFunction[(Aggregate, Command), Future[Event]]))
...
```
Behind the scenes we where lifting all these ParticalFunctions to `PartialFunction[T, Future[List[Event]]]))` composing them with `orElse` and building a huge PF that represented the Behavior.

This first version worked as expected and gave us the possibility to declare the behavior of an aggregate they way we thought we should do. However, it rapidly grew to huge blocks of PartialFunctions that were hard to follow and have a good picture of all possible cases we were covering.

At some point this same API was refactored to use the Magnet Pattern. As such we could by-pass the type erasure and have less method variations. We end up with basically 2 kinds of command handlers and a bunch of Magnets. 

```scala
def handleCommand(handler: PartialFunction[Command, EventMagnet]))
def handleCommand(handler: PartialFunction[(Aggregate, Command), EventMagnet]))
```
And the final shape of the DSL was something more or less like the following:

```scala
// pseudo-scala code, won't compile ;-)
behaviorFor[MyAggregate]
  .whenCreation { it => 
    it.acceptsCommand {
      case CreationCmd => CreatedEvt
    }
    .appliesEvent { 
      case CreatedEvt => MyAggregate
    } 
  }
  .whenUpdating { it =>
      it.acceptsCommand { 
          case (agg, cmd: SomeCmd) => SomeEvt(cmd.blabla)
      }
      .appliesEvent {
        case (agg, SomeEvt) => agg.copy(...)
      }
  }
```

## Declarative and Composable Behavior

After collecting feedback from different kind of people (Scala devs, Java devs, DDD experts) we came to the conclusion that the API was actually quite heavy, too technical, boring and hard to understand. Specially when the aggregate had to react to several commands and events. 

We start to search for something that could be composable and clearly express the actions (command handlers and event handlers) that were applicable to a given state of the aggregate. 

Since we were already using the term "Behavior" it seemed to us a good idea to read the definition of the word to see if we could get some expiration. 

The English Webster Dictionary defines the worlds **Bevahior** as such...
> "The way in which an animal or person behaves in response to a particular situation or stimulus"
>  - Webster Dictionary 

If we bring it to a CQRS context, we can say that the behavior of a model (write-model) is the sum of all actions it will perform given a stimulus (commands / events) and a particular situation (model current state).

So, depending on the situation we must have a different set of actions. It goes without saying that we may perform the same action in more than one situation which bring us to the inception that actions must be reusable. 

If the **situation** is the state of the aggregate, we must start from a decision structure that goes from state to actions. It was clear that this had to have the shape of a pattern matching. Therefore we define a **PartialFunction** from State => Actions. 

```scala
type Behavior = PartialFunction[State[A], Actions[A]])
```
`State` works like Option, it can be Uninitialized or Initialized. `Actions` carries the Command Handlers and Event Handlers.

And an `Action` is defined as following:

```scala
// using example from Fun.CQRS sample project
def createLottery(lotteryId: LotteryId) = 
  actions[Lottery]
    .handleCommand {
      cmd: CreateLottery => LotteryCreated(cmd.name, metadata(lotteryId, cmd))
    }
    .handleEvent {
      evt: LotteryCreated => Lottery(name = evt.name, id = lotteryId)
    }
```

The Behavior can now be define as a PartialFunction from State to Actions as follow:

```scala
def behavior(lotteryId: LotteryId): Behavior[Lottery] = {

    case Uninitialized(id) => createLottery(id) 

    case Initialized(lottery) if lottery.hasWinner => lottery.rejectAllCommands 

    case Initialized(lottery) if lottery.hasNoParticipants =>
      lottery.canNotRunWithoutParticipants ++
        lottery.acceptParticipants

    case Initialized(lottery) if lottery.hasParticipants => 
      lottery.rejectDoubleBooking ++
        lottery.acceptParticipants ++
        lottery.removingParticipants ++
        lottery.runTheLottery
  }
```
Actions can be composed and reused as we can see from the `acceptParticipants` actions. We may accept participants in two situations (or states), when if we don't have yet any participant and when we have already some but the lottery is not run yet.

(work in progress...)
