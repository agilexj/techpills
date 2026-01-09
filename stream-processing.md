# Key Terminology

## Event
An event is a single occurrence or record in a stream. Each event is typically characterized by its type and associated data.

## State
State refers to the information maintained across events during stream processing. It can be either **stateful** (maintaining information across events) or **stateless** (not retaining information).

## Windowing
Windowing is a technique used to group events into manageable chunks for processing. There are various types of windows, including:
* **Time-based Windows:** Events are grouped based on time intervals.
* **Count-based Windows:** Events are grouped based on a specified count of events.

## Time and Ordering
In stream processing, ordering of events is important, especially when dealing with time-sensitive data. Events may arrive out of order, so systems must manage this effectively. A common technique involves using timestamps

# Event Driven Architecture

## Challenges in Event-Driven Architecture
While EDA has many benefits, there are also challenges to consider:
* **Event Ordering:** Maintaining the order of events can be challenging, especially in distributed systems.
* **Event Schema Evolution:** As systems evolve, so do event schemas, which can lead to compatibility issues.
> **Warning:** Monitor and manage event schemas to avoid breaking changes that could disrupt consumers. Consider using a schema registry for better schema management.

## Event Storage and State Management
In an event-driven architecture, managing the state of events is crucial, particularly for applications that require high reliability. Events may need to be stored for various reasons, including:

* **Replayability:** Storing events allows systems to replay them for debugging or analysis.
* **State Restoration:** In case of failures, stored events can help restore the system to a previous state.

For state management, event sourcing is a common pattern where the state of an application is derived from a sequence of events.

## Best Practices for Event-Driven Architecture
To effectively implement EDA, consider the following best practices:

* **Ensure Idempotency:** Consumers should handle events in such a way that processing the same event multiple times does not lead to incorrect results.
* **Use Schema Registries:** Maintain a schema registry to manage the evolution of event schemas over time.
* **Implement Monitoring:** Monitor events and their processing to swiftly identify issues in the system.