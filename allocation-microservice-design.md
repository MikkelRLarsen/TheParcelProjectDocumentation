# Allocation Microservice Design

## Overview

The **Allocation microservice** is responsible for allocating parcels to
terminals based on routing decisions. It acts as an intermediary between
the **Routing service** and the **Terminal service**, ensuring that
parcels are only assigned when the target terminal has available
capacity.

If capacity is not available, the parcel request is queued and retried
when capacity becomes available.

The system is designed using an **event-driven architecture** to avoid
polling and unnecessary resource usage.

------------------------------------------------------------------------

# Responsibilities

The Allocation service is responsible for:

-   Receiving allocation requests from the Routing service
-   Attempting to allocate parcels to terminals
-   Queueing allocation requests when terminal capacity is unavailable
-   Listening for capacity updates from the Terminal service
-   Retrying queued allocations when capacity becomes available
-   Ensuring safe concurrent processing

------------------------------------------------------------------------

# Allocation Flow

## 1. Parcel Created

1.  A parcel is created in the **Parcel Service**
2.  A `ParcelCreatedEvent` is published
3.  The **Routing Service** calculates the optimal route

------------------------------------------------------------------------

## 2. Routing Requests Allocation

After calculating the next terminal in the route, the Routing service
sends an allocation request:

    AllocateParcel(parcelId, terminalId)

The Allocation service then attempts to allocate the parcel.

------------------------------------------------------------------------

## 3. Immediate Allocation Attempt

The Allocation service attempts to allocate the parcel at the requested
terminal.

    TryAllocate(parcel, terminal)

Two outcomes are possible:

**Success**

    Parcel allocated → process complete

**Failure (Terminal full)**

    Parcel is added to queue for the terminal

------------------------------------------------------------------------

# Queueing Strategy

If a terminal has no available capacity, the allocation request is
queued.

The queue structure is organized **per terminal**:

    Dictionary<TerminalId, PriorityQueue<AllocationRequest>>

Reasons:

-   Prevents terminals from blocking each other
-   Allows efficient queue processing
-   Supports prioritization if needed

The queue is implemented as a **Priority Queue (MinHeap)**.

Possible priority rules:

-   Timestamp (FIFO behavior)
-   Parcel priority
-   Delivery urgency

------------------------------------------------------------------------

# Event-Driven Retry Mechanism

The system avoids polling by relying on **capacity update events** from
the Terminal service.

When capacity changes, the Terminal service publishes:

    TerminalCapacityUpdatedEvent
    {
        TerminalId
        CapacityFreed
    }

This event indicates that new parcel slots are available.

------------------------------------------------------------------------

# Processing Capacity Updates

When the Allocation service receives a `TerminalCapacityUpdatedEvent`,
it processes queued allocation requests.

The service attempts to allocate up to `CapacityFreed` parcels from the
terminal's queue.

Example logic:

``` csharp
for (int i = 0; i < evt.CapacityFreed; i++)
{
    if (!_queue.TryDequeue(evt.TerminalId, out var request))
        break;

    await TryAllocate(request);
}
```

This ensures that each newly available capacity slot is used to allocate
one queued parcel.

------------------------------------------------------------------------

# Concurrency Control

Multiple events may arrive simultaneously. To prevent race conditions,
queue processing is protected using a **per-terminal semaphore lock**.

    ConcurrentDictionary<TerminalId, SemaphoreSlim>

Example:

``` csharp
var terminalLock = terminalLocks.GetOrAdd(
    evt.TerminalId,
    _ => new SemaphoreSlim(1,1));

await terminalLock.WaitAsync();

try
{
    // Process queue
}
finally
{
    terminalLock.Release();
}
```

This ensures:

-   Only one allocation process per terminal at a time
-   No double allocation
-   Consistent queue ordering

------------------------------------------------------------------------

# Sequential Allocation

Allocation requests are processed **sequentially**, not in parallel.

Reason:

Terminal capacity represents a **limited resource**, and sequential
processing ensures:

-   deterministic allocation order
-   correct capacity usage
-   simpler concurrency management
-   reduced risk of race conditions

Example:

    CapacityFreed = 3

    Parcel A → allocated
    Parcel B → allocated
    Parcel C → allocated

Parallel allocation was intentionally avoided.

------------------------------------------------------------------------

# Why Event-Driven Instead of Polling

Polling was avoided because it would:

-   waste CPU resources
-   introduce unnecessary delays
-   increase system complexity

Event-driven processing ensures:

-   immediate reaction to capacity changes
-   efficient resource usage
-   cleaner microservice boundaries

------------------------------------------------------------------------

# Final Architecture Flow

    Parcel Service
          │
    ParcelCreatedEvent
          │
    Routing Service
          │
    RouteCalculated
          │
    Allocation Service
          │
    TryAllocate
     ┌────┴─────┐
    Success    Fail
     │           │
    Done      Queue
                  │
    TerminalCapacityUpdatedEvent
                  │
    ProcessQueuedAllocations
                  │
    Allocation retried

------------------------------------------------------------------------

# Key Design Decisions

  Decision                        Reason
  ------------------------------- -----------------------------------------
  Event-driven capacity updates   Avoid polling
  Per-terminal queues             Prevent cross-terminal blocking
  Priority queue                  Fair ordering and future prioritization
  Sequential allocation           Avoid race conditions
  Semaphore per terminal          Ensure safe concurrency

------------------------------------------------------------------------

# Future Improvements

Potential improvements include:

-   Dead-letter queue for failed allocations
-   Retry limits
-   Parcel prioritization
-   Persistent queue storage
-   Monitoring and metrics

------------------------------------------------------------------------

# Summary

The Allocation service ensures reliable parcel distribution by
combining:

-   **event-driven architecture**
-   **queue-based retry handling**
-   **per-terminal concurrency control**

This design ensures that parcels are allocated efficiently while
maintaining system consistency and scalability.
