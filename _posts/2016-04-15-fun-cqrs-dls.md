
##  A CQRS Behavior DSL

When we started the development of Fun.CQRS we had one major goal in mind. We wanted a comprehensive API that we could reuse in different internal projects. The API must be minimalistic and only expose the basic operations that all CQRS / ES need. It should also be flexible enough to accommodate use cases that we didn't think about in the beginning. 

The choice for **Akka** as a backend was easy. Our team had already experience with it and **Akka Persistence** had almost all the machinery we needed for Event Sourcing. The only missing piece at the time was the **Query** capability. For that we had to develop our own system to query and stream events to our **Projections**.  This was later replaced by the new experimental **Akka Persistence Query** model.

**Akka** gave us **Event Sourcing**, so we need to something to help us modeling our domain in terms of **Commands** and **Events**. And of course, using our favorite Scala style of modeling, i.e: case classes and functions.

Defining the basic operations we needed was easy, the challenge was how to make it work inside **Akka** in a transparent way, not because we were not happy with **Akka**, on the contrary, but because our feeling was that we should abstract out everything that was not directly related with the business domain.

We had come up the following specification:  

* Whatever it happens, everything must be expressed in terms of **Commands** and **Events**.   
* Every single aggregate has at least two phases, whether it is initialized whether it is not.  
Uninitialized aggregates need a **Factory** that takes the role of constructor. It accepts **Commands**, validates it and emits  one or more **Event** (initially we thought that we should generate only one Event at construction type, we changed our mind later on).

The flow is well known. 

 1. A **Command** is sent to an existing **Aggregate** or to its **Factory**
 - **Command** is validated, if valid one or more **Events** will be emitted
 - **Events** are persisted (but this must be transparent to the user)
 - **Events** are applied
  * if at construction time, a new **Aggregate** is instantiated
  * if it's an existing **Aggregate**, events are applied to producing a new state
 - **Aggregate** is ready to accept the next **Command**

Although in the majority of the cases we will produce one single Event, we need to support the possibility to produce more than one Event in some situations.
 
 Although in the majority of the cases we will be validating a Command solely against the current state of the aggregate, in some situations we need to run a query or call a external service. 
 
Given the specifications above we came up with the following functions:

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

### Behavior definition
> "The way in which an animal or person behaves in response to a particular situation or stimulus"
>  - Webster Dictionnary 

Behavior has a state and functions that reflects the Actions (command handlers and event handlers).


If we bring it to a CQRS context, we can say that the behavior of a model (write-model) is the sum of all actions it will perform given a stimulus (commands) and situation (model current state).
