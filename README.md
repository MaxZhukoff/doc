# **Example of using the Saga pattern implementation**
<sub><sub>[in Russian](https://github.com/MaxZhukoff/saga-examples/tree/master/simple-saga-demo)</sub></sub>

## **Introduction to the Sagas**

One drawback of the microservice architecture concerns the execution of transactions that involve multiple services. In a monolithic architecture, the DBMS (Database Management System) fully provides the functionality of *ACID transactions*. However, in a microservices architecture which uses the *"Database per service"* pattern, a more elaborate transaction processing mechanism is required.

And the Saga pattern, as a way to organize transactions in a distributed system, is a good solution in such a situation. The definition of Saga is as follows – *a sequence of local transactions coordinated by asynchronous messages*. I.e. each service, after completing its part of the distributed transaction and recording it in its local database, publishes an event, on the basis of which the following Saga participants.

In *ACID transactions*, in case of failure, the *rollback* command is executed automatically to ensure atomicity. However, when using the Saga pattern, you need to write corresponding compensating transactions that should undo the changes already made. This will be illustrated with examples.

The Saga pattern can be implemented in two ways: *orchestration* and *choreography*. In the library's implementation, a choreographic approach is used, where there is no central coordinator issuing commands to participants. Instead, participants subscribe to each other's events and react accordingly. Choreographic Sagas can also be called dynamic because there is no centralized place to see which services are participants in a particular distributed transaction, and because the Saga itself can constantly change.

## **Implementation of Saga in the library**

You can now take a look at how this is implemented in the library.  
Interaction with Sagas is done using `SagaManager`. By default, all the tools for interacting with Sagas are enabled, but if you want to create your own implementation, you can disable the built-in Saga pattern implementation in the configuration by setting: `event.sourcing.sagas-enabled=false`.

To start a Saga, you need to create the first step of the Saga. You can do this by calling the `launchSaga(sagaName, stepName)` method on the `SagaManager`, providing `sagaName` as the name of the process and `stepName` as the name of the current step. After executing this method, you will obtain a `SagaContext`, which represents the result of the Saga step, and you can pass this context when updating the aggregate.
```kotlin
//Initialization of SagaContext by SagaManager
val sagaContext = sagaManager
    //Initialize first step, sagaName="SAGA_EXAMPLE", stepName="First step"
    .launchSaga("SAGA_EXAMPLE", "First step")
    .sagaContext //Get initialized sagaContext

//Pass SagaContext to the method for creating/updating an aggregate
//As a result, it will be stored in the event
val event = someEsService.create(sagaContext) {
    it.someMethod(params...)
}
//or
val event = someEsService.update(aggregateId, sagaContext) {
    it.someMethod(params...)
}
```
Inside the `SagaManager`, a `SagaContext` is created - a class that contains a `ctx` map where the key is `sagaName` and the value is metadata about the previous step. This metadata includes:
- `stepName`,  which, like `sagaName`, is set by the user
- `sagaStepId` – the identifier of the Saga step.
- `sagaInstanceId` – the identifier of the Saga itself, which will be common to a particular distributed transaction
- `prevStepsIds` – identifiers of previous Saga steps

`SagaContext` is passed along with the event that the next service in the Saga execution chain subscribes to. In that service, you can extract the previous context and continue Saga execution.  
All of this is done through `SagaManager` by calling `withContextGiven(SagaContext)` method, followed by `performSagaStep`, which takes the same parameters as `launchSaga`:
```kotlin
val sagaContext = sagaManager
    .withContextGiven(event.sagaContext) //Extract metadata from the previous event
    .performSagaStep("SAGA_EXAMPLE", "Second step")
    .sagaContext
```
To track the sequence of events in a Saga, the `SagaContext` contains `prevStepsIds/correlationId` - identifiers of the previous or one previous Saga step.  
The following picture roughly demonstrates how this metadata is passed from one service to another.  
![SagaProcessing](images/SagaProcessing.PNG)

You can view the code that is made from this image _[here](https://github.com/MaxZhukoff/saga-examples/tree/master/simple-saga-demo)_. _But it's a bit simplified there._  
Here's how the execution scenarios of the Saga based on events look in this example:

![SagaExample](images/SagaExample.PNG)

The successful scenario is denoted in green, and the unsuccessful one is in red. If one of the Saga transactions fails, it is necessary to initiate the logic of **compensating** transactions to maintain consistency in the system.

In the library, compensating transactions are defined in the same way as regular ones. That is, in case of an error, you need to create an event that the services, which were previous in the transaction chain, will act upon.

For example, let's consider what will happen after an error occurs in the `"flights"` service.  
First, the _`FlightReservationCanceledEvent`_ event is created. After that, the subscriber of the `"payment"` service - [_`PaymentFlightSubscriber`_](https://github.com/MaxZhukoff/saga-examples/blob/master/simple-saga-demo/src/main/kotlin/ru/quipy/saga/simple/payment/subscribers/PaymentFlightSubscriber.kt) - will roll back the changes made earlier in this service. Then, the process will return to the service that started this Saga, which will record the fact that the entire process has failed.  
I.e. you, as a user, build the necessary business process yourself, including the logic of _compensating transactions_.

### **_Parallel processes_**

If your business logic requires Saga to execute not only sequentially, but also in **parallel**, meaning that you need to wait for the completion of several previous steps to continue with a particular step, you can pass multiple `sagaContexts` to the `witchContextGiven()` method by combining them using the **"+"** operator. You can see an example of Saga with parallel steps by following this _[link](https://github.com/MaxZhukoff/saga-examples/tree/master/aggregation-saga-demo)_.  
Here's what the execution scenario of this Saga looks like:  
![AggregationSagaExample](images/AggregationSagaExample.PNG)

In the example, it is necessary to wait for two events - `Service2ProcessedEvent` and `Service3ProcessedEvent` in the account service (`service1`). One way to achieve this is to create a projection/view to store events. [Here](https://github.com/MaxZhukoff/saga-examples/blob/master/aggregation-saga-demo/src/main/kotlin/ru/quipy/saga/aggregation/service1/subscribers/Service2Service3Subscriber.kt), a projection named `aggregation-example` is created, where the id corresponds to the Saga's id. It contains certain information from two events that are necessary to continue the Saga. When one of the events arrives, it checks if a projection with the Saga's id exists. If it doesn't exist, a new projection is created and the information about the first event is stored. However, in the case where the projection already exists, the execution of the Saga continues based on the data from this projection.  
Here's how the context is created based on two previous `sagaContexts`
```kotlin
val sagaContext = sagaManager
    .withContextGiven(prevSagaContext_1 + prevSagaContext_2)
    .performSagaStep("SAGA_EXAMPLE", "finish saga")
    .sagaContext
```
For such an event created based on two previous steps, the `prevStepsIds` will accordingly contain two Saga step IDs.

#### **_Nested Sagas_**

The `ctx` map in `SagaContext` was intentionally included to allow the creation of **nested Sagas** using this feature.  
If you need to separate multiple steps within a single saga, i.e. create a **nested saga**, you can do it as shown in this _[example](https://github.com/MaxZhukoff/saga-examples/tree/master/nested-saga-demo)_.  
Here's what the execution scenario of this Saga looks like:  
![NestedSagaExample](images/NestedSagaExample.PNG)

In this case, the nested Saga is also parallel, but that's not a prerequisite. The idea is to compartmentalize subprocesses within a larger business process.  
In this scenario, the main Saga will include all the steps, but the nested Saga will contain only the two steps highlighted in red.  
The creation of the nested Saga `NESTED_SAGA_EXAMPLE` and the parallel continuation of the main `SAGA_EXAMPLE` takes place in [`Service2`](https://github.com/MaxZhukoff/saga-examples/blob/master/nested-saga-demo/src/main/kotlin/ru/quipy/nested/service2/subscribers/Service1Subscriber.kt). The next step takes place in [`Service3`](https://github.com/MaxZhukoff/saga-examples/blob/master/nested-saga-demo/src/main/kotlin/ru/quipy/nested/service3/subscribers/Service2Subscriber.kt).

### **_Simplified version_**

In the library, there is the possibility to use a **simplified approach** to implement the Saga pattern, without using `SagaManager`. For this purpose, `SagaContext` contains three additional values in addition to `ctx`:
 - `correlationId` – means the same as `sagaInstanceId` from `ctx`
 - `currentEventId` – means the same as `sagaStepId` from `ctx`
 - `causationId` – means the same as `prevStepsIds` from `ctx`, but can only contain one value

If you use this simplified approach, the Saga and its steps will not have names. Also, you won't be able to create nested Sagas or parallel steps.  
To use this approach you just need to pass the `SagaContext` of the previous event when updating the aggregate.  
You can see an example of such a Saga _[here](https://github.com/MaxZhukoff/saga-examples/tree/master/default-saga-demo "This is a remodeled first example")_.

## **Saga projections**

Currently, to track the entire execution process of a Saga, you need to contact each participating service. However, using projections, you can gather information about Sagas in one place. This allows you to monitor the state of Sagas, retrieve all steps for a `sagaInstanceId`, track the time it took to process each step, and determine in which service an error occurred.

To understand how projections are set up, it's helpful to have some knowledge of how certain processes inside the library related to Sagas work.

For local storage of information about Sagas, the `SagaStepAggregate` is used. This aggregate has 4 events:
- `SagaStepLaunchedEvent` – indicates that the first Saga step has been **initiated** but has not yet been processed
- `SagaStepInitiatedEvent` – indicates that a subsequent Saga step has been **initiated** but has not yet been processed
- `SagaStepProcessedEvent` – indicates that the Saga step has been **processed** and stored in the `EventStore`
- `DefaultSagaProcessedEvent` –  indicates that the default Saga step has been processed

All these events are stored in the `sagas` table. Each service/bounded context will have its own table for this purpose.

The library also includes an *event-stream* called `SagaEventStream` for working with Sagas. It's necessary to keep track of whether a Saga event has been successfully processed and stored in the database. This *event-stream* subscribes to events from **all** aggregates and looks for *meta-information* related to Sagas in them. It then sends commands to the `SagaStepAggregate`, which publishes either `SagaStepProcessedEvent` or `DefaultSagaProcessedEvent` events.

So `SagaManager` is used to record the fact that a Saga step has started processing, while `SagaEventStream` is used to determine when the event has been successfully processed.

It's important to note that in the simplified version of using Sagas, the `SagaStepLaunchedEvent` and `SagaStepInitiatedEvent` events are not published because `SagaManager` is not used. Instead of that, only one event is published - `DefaultSagaProcessedEvent`.

In the library, there is already a ready-made service from the `tiny-event-sourcing-sagas-projections` module for tracking Sagas.  
This service subscribes to events from the local Saga aggregates `SagaStepAggregate` and creates tables `sagas-projections` for the extended version and `sagas-default-projections` for the simplified version. This service allows you to keep track of all the Sagas in the system in one place.

In the projection, a separate document is created for each Saga, containing its `sagaInstanceId`, `sagaName`, and all its steps - `sagaSteps`. In `sagaSteps`, in addition to `sagaStepId`, `prevStepsId` and `stepName`, there is information about when the step processing started, when it was successfully completed and the name of the event associated with this step. The entire sequence of steps can be traced using `sagaStepId` and `prevStepsId`. Additionally, a simple sorting by `sagaStepId` and `prevStepsId` is implemented.

To create your own service for tracking Sagas, you first need to determine what will be stored in the projections.  
(*String* is used everywhere for easier representation of this data in the database, but it's not mandatory).
```kotlin
@Document("sagas-projections")
data class Saga(
    @Id
    val sagaInstanceId: String, //UUID
    val sagaName: String?,
    val sagaSteps: MutableList<SagaStep> = mutableListOf()
)

data class SagaStep(
    val stepName: String,
    val sagaStepId: String, //UUID
    val prevStepsId: Set<String> = setOf(), //UUID
    //Step processing start time (in milliseconds)
    val initiatedAt: String, //Long
    //The time when the step was processed and stored in the database (in milliseconds)
    var processedAt: String? = null, //Long
    var eventName: String? = null
)
```
Second, you need to create a subscriber for events from `SagaStepAggregate`. [At the beginning of the chapter there is a list and descriptions of all events](#Saga-projections). You should store the necessary information in the subscriber.

An example for `SagaStepLaunchedEvent`:
```kotlin
subscriptionsManager.createSubscriber(SagaStepAggregate::class, "local-sagas::sagas-projections") {
        `when`(SagaStepLaunchedEvent::class) { event ->
            val sagaName = event.sagaName
            val stepId = event.sagaStepId.toString()
            val stepName = event.stepName

            val saga = Saga(event.sagaInstanceId.toString(), sagaName)

            val newStep = SagaStep(
                stepName,
                stepId,
                event.prevSteps.map { it.toString() }.toSet(),
                event.createdAt.toString()
            )
            insertNextStep(saga.sagaSteps, newStep)

            sagaProjectionsRepository.save(saga)
        }
    }
}
```
In the example above, information about the Saga and its step is extracted from the event and stored in the database.  
For `SagaStepInitiatedEvent` the behavior will be almost the same as for `SagaStepLaunchedEvent`.  
For `SagaStepProcessedEvent`, it is sufficient to update the already saved Saga step:
```kotlin
`when`(SagaStepProcessedEvent::class) { event ->
    //Find the Saga projection that requires updating the step
    val saga = sagaProjectionsRepository.findById(event.sagaInstanceId.toString()).get()

    //Extract the information from the event
    val sagaName = event.sagaName
    val stepId = event.sagaStepId.toString()
    val stepName = event.stepName

    //Update the Saga step
    val sagaStep = saga.sagaSteps.find { it.sagaStepId == stepId }
    sagaStep?.processedAt = event.createdAt.toString()
    sagaStep?.eventName = event.eventName
}
```