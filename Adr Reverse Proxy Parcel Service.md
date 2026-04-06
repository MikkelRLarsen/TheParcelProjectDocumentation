# Architecture Decision Record: Introduce Reverse Proxy for Parcel Service

## Owner

Mikkel Rosendahl Larsen

## Date

06-04-2026

## Status

Proposed

## Context

TheParcelApplication is an event-driven parcel routing and allocation system using multiple microservices: ParcelService, RoutingService, AllocationService, and TerminalService. Currently, WebApp / external clients call ParcelService directly via its API, and other services communicate via event-driven messaging (Redis Pub/Sub).

To simplify external access and create a unified entrypoint, the team considers introducing a Reverse Proxy in front of ParcelService. This would allow all external API calls to go through a single gateway, centralizing routing, authentication, TLS, and monitoring.

## Decision

Introduce a Reverse Proxy container that sits in front of ParcelService (and optionally other Services) to act as the single entrypoint for all external API calls. The proxy will:

* Route incoming HTTP requests from WebApp / external users to the correct microservice (ParcelService, etc.)
* Handle TLS termination and authentication if required
* Support future scalability and expansion of new features and services

Internal service-to-service communication (event-driven via Redis Pub/Sub) remains unchanged and does not pass through the Reverse Proxy.

## Rationale

* **Single entrypoint** simplifies API consumption for clients
* **Flexibility**: future services can be exposed via the same proxy without changing client code
* **Monitoring**: centralizes logging, metrics, and request tracing

## Consequences

### Positive

* Easier external client integration
* Centralized security and observability
* Simplifies future API versioning and routing
* Supports load balancing and high availability, if changing from K8 to other orchestration

### Negative

* Adds an additional network hop for requests
* Introduces potential single point of failure if proxy is not highly available
* Requires operational effort to configure and maintain proxy (e.g., Nginx, HAProxy, Traefik, YARP)

## Alternatives Considered

### Keep direct access to ParcelService

Rejected because:

* Clients need to know multiple service endpoints
* Harder to scale with new endpoints

### Use API Gateway instead of Reverse Proxy

Considered but postponed because:

* Current requirements only need routing, since Auth is not a part of project scope
* Full API Gateway features (rate-limiting, auth, caching) can be added later if needed
* More expirienced with Reverse Proxies

## Example Flow

1. User calls `/parcel` endpoint on Reverse Proxy
2. Reverse Proxy routes request to ParcelService API
3. ParcelService creates new parcel and publishes `ParcelCreated` event
4. Event-driven flow continues with RoutingService, AllocationService, and TerminalService as before

## Notes

* Reverse Proxy should be containerized and deployed alongside ParcelService in Kubernetes
* Keep internal event-driven architecture unaffected by proxy introduction
