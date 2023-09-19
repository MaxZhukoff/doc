## . . . Release
#### .. .. , 2023
### What's new:

Added Saga pattern implementation. The Saga pattern can be implemented using library with a choreographic approach.  
- Added `SagaManager` class that simplifies the initiation and management of Saga transactions.
- Each Saga transaction is associated with `SagaContext` that holds essential metadata, including step names, step IDs, Saga instance IDs, and previous step IDs. `SagaContext` can be created using the `SagaManager`:
    ```kotlin
    val sagaContext = sagaManager
        .launchSaga("SAGA_EXAMPLE", "First step")
        .sagaContext
    ```
- Added ability to pass `SagaContext` to `create` and `update` methods in `EventSourcingService`:
    ```
    someEsService.create(sagaContext) {
        it.someMethod(params...)
    }
    someEsService.update(aggregateId, sagaContext) {
        it.someMethod(params...)
    }
    ```
- `SagaContext` is stored in an event that can be retrieved in the subscriber:
    ```kotlin
    val sagaContext = sagaManager
        .withContextGiven(event.sagaContext)
        .performSagaStep("SAGA_EXAMPLE", "Second step")
        .sagaContext
    ```
- The "+" operator can be used to merge multiple SagaContexts: `withContextGiven(prevSagaContext_1 + prevSagaContext_2)`
- Nested Sagas are possible.
- A simplified approach to implementing Sagas is available, allowing for Saga coordination without using `SagaManager`.
- Introduced the concept of **Saga projections** for tracking the execution process of Sagas. Added the `tiny-event-sourcing-sagas-projections` module.
- Added [documentation](https://github.com/andrsuh/tiny-event-sourcing#introduction-to-the-sagas) for implementing Saga in the library.
