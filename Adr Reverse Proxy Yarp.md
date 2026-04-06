# Architecture Decision Record: Reverse Proxy Implementation Choice (YARP vs Kubernetes-native)

## Owner

Mikkel Rosendahl Larsen

## Date

06-04-2026

## Status

Proposed

## Context

TheParcelApplication uses multiple microservices (ParcelService, RoutingService, AllocationService, TerminalService). Currently, WebApp and external clients call ParcelService directly. Internal communication remains event-driven via Redis Pub/Sub.

To simplify external access and create a unified entrypoint, the team considers introducing a Reverse Proxy. Two main options were evaluated:

1. **YARP (Yet Another Reverse Proxy)** – a .NET-based reverse proxy.
2. **Kubernetes-native solutions** – e.g., NGINX Ingress Controller, Traefik, or similar.

Requirements include:

* Unified external entrypoint
* Flexible routing between services
* Deployment within Kubernetes

## Decision

Adopt **YARP** as the Reverse Proxy solution. YARP will run as a containerized service in Kubernetes in front of ParcelService (and optionally other services).

Responsibilities of the proxy:

* Route incoming HTTP requests to the appropriate microservice endpoints
* Handle TLS termination and authentication if required
* Support future extensions, such as dynamic route configuration or complex routing logic

Internal service-to-service communication via Redis Pub/Sub remains unaffected.

## Rationale

* **.NET ecosystem alignment**: Team has strong .NET expertise, making YARP easier to configure, extend, and maintain.
* **Dynamic routing flexibility**: YARP supports complex routing scenarios, request transformations, and middleware integration directly in C# code.
* **Future extensibility**: Adding new microservices or API features can be done without changing client code or Kubernetes configuration.
* **Avoids Ingress limitations**: While Kubernetes-native ingress controllers provide basic routing and TLS, advanced transformations or dynamic per-request routing require annotations or external CRDs, adding complexity.

## Consequences

### Positive

* Full control over routing logic and middleware in .NET
* Easy integration with existing monitoring/logging stack
* Simplifies development of dynamic routes or feature toggles
* Leverages existing team expertise

### Negative

* Requires operational effort to configure, deploy, and maintain the proxy
* Adds an additional network hop for external requests
* Single point of failure if proxy is not configured for high availability
* Less “K8-native” – scaling and TLS automation require explicit configuration

## Alternatives Considered

### Kubernetes-native Ingress/Proxy

Pros:

* Native integration with Services, ConfigMaps, and cert-manager
* Auto-scaling and high availability managed by Kubernetes
* Standard solution for many K8 deployments

Cons:

* Limited dynamic routing without custom annotations or middleware
* Less flexibility for request transformation in .NET
* Adds complexity for advanced scenarios (e.g., conditional routing, header rewriting, custom metrics)

### Keep direct access to ParcelService

Rejected because:

* Clients must manage multiple endpoints
* Harder to evolve API without breaking clients
* No centralized logging, TLS, or authentication entrypoint

## Example Flow

1. User calls `/parcel` endpoint on YARP Reverse Proxy
2. YARP routes request to ParcelService API
3. ParcelService publishes `ParcelCreated` event
4. Event-driven flow continues with RoutingService, AllocationService, TerminalService

## Notes

* YARP will be containerized and deployed in Kubernetes alongside services
* Internal Redis Pub/Sub communication remains unchanged
* Future: TLS termination can be integrated with cert-manager via Kubernetes secrets, but routing and transformation remain under YARP control
