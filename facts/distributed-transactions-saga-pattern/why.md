Why not just lock the databases across all 3 services using traditional **Two-Phase Commit (2PC)** protocols?

Because 2PC is the arch-enemy of Scalability. 

During a Two-Phase commit, the `Order Database` and `Inventory Database` would have to lock those rows strictly, waiting over the network for the `Payment API` to confirm everything before finally committing. If the Payment API takes 15 seconds to reply, those inventory rows are locked for 15 seconds. If Black Friday hits, your system comes to a grinding halt.

### Implementing Saga
You can implement the Saga pattern in two primary ways:

1. **Choreography**: (Event-Driven) Every service publishes an event to a Message Broker like Kafka/RabbitMQ. The next service listens, does its job, and publishes another event. If something fails, it publishes a `PaymentFailed` event, and the other services listen for that to undo their work. Excellent for small systems but hard to conceptualize.
2. **Orchestration**: (Command-Driven) An overarching "Orchestrator" service (like AWS Step Functions or Netflix Conductor) tells each service exactly what to do. If a step fails, the Orchestrator explicitly commands the specific services to run their rollback endpoints. Better for complex enterprise workflows.
