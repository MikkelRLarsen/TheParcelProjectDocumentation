# Architecture Decision Record: Separation of Routing, Allocation & Termnial services
## Owner
Mikkel Rosendahl Larsen

## Date
05-04-2026

## Status
Accepted

## Context
The system is an event-driven parcel routing and allocation platform inspired by logistics domain systems. The architecture follows a microservices approach with services such as RoutingService, AllocationService, and TerminalService.

A design decision needed to be made regarding whether AllocationService should exist as a separate microservice or be merged into TerminalService.

The system includes the following requirements:
- Routing must determine optimal geographical or network-based paths.
- Multiple terminals may exist within a region.
- Load balancing across terminals is required.
- The system should support partitioning and scaling per region.
- Eventual consistency is acceptable, including retry mechanisms.
- Terminal capacity may change dynamically, introducing race conditions.

## Decision
AllocationService will be implemented as a separate microservice responsible for:
- Selecting the least loaded terminal within a given region
- Managing queueing and prioritization strategies
- Handling retries in case of terminal rejection
- Acting as a distributed scheduling and coordination layer

RoutingService will:
- Determine the optimal region or set of candidate terminals using graph-based algorithms

TerminalService will:
- Handle execution, including accepting or rejecting parcels based on real-time capacity

## Rationale
This separation aligns with the principle of separation of concerns:
- RoutingService handles global path decisions
- AllocationService handles local load balancing decisions
- TerminalService handles execution

The decision is justified by the following factors:
- Need for regional partitioning and scalability
- Multiple terminals per region requiring load balancing
- Dynamic prioritization and queue management
- Support for retry and eventual consistency patterns
- Avoidance of overloading individual terminals

## Consequences

### Positive
- Improved scalability through partitioned AllocationService instances
- Clear separation of responsibilities
- Flexibility to evolve allocation strategies independently
- Better support for advanced features such as prioritization and SLA handling
- Natural implementation of retry logic and eventual consistency
- Posibility to change queue logic without impacting TermnialService

### Negative
- Increased system complexity due to additional microservice
- Additional network latency between services
- More complex state management and potential for inconsistency
- Increased operational overhead (deployment, monitoring, CI/CD)

## Alternatives Considered

### Merge Allocation into TerminalService
Rejected because:
- Would couple load balancing logic with execution
- Makes scaling and partitioning more difficult
- Reduces flexibility in evolving allocation strategies

### Perform Allocation in RoutingService
Rejected because:
- Mixes global routing concerns with local load balancing
- Reduces clarity of responsibilities
- Makes routing logic more complex and harder to maintain

## Example Flow
1. ParcelCreated event is published
2. RoutingService determines a sequence of region and keeps track via SAGA.
3. AllocationService selects least loaded terminal in given region.
4. TerminalService attempts to accept parcel
5. If rejected, AllocationService retries with least active queued termnial in region

## Notes
AllocationService should only operate within the scope defined by RoutingService (e.g., region or candidate terminals) to avoid overlapping responsibilities.
