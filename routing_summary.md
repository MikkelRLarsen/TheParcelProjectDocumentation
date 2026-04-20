# Routing & Terminal Architecture (Exam Summary)

## Overview
System models a logistics network in Denmark using:
- Pickup Points
- Distribution Centers (DC)
- Hubs

Built with microservices:
- Routing Service
- Allocation Service
- Terminal Service

---

## Terminal Types

### Pickup Point
- Customer interaction (send/receive packages)
- Connected to 1–2 DCs

### Distribution Center (DC)
- Regional sorting
- Connects to:
  - Pickup Points
  - Other DCs
  - Hubs

### Hub
- High-capacity, long-distance transport
- Connects to:
  - All DCs in region
  - Other hubs

---

## Network Structure

Typical flow:
Pickup → DC → (DC/Hub) → DC → Pickup

Common patterns:
1. Local: Pickup → DC → Pickup
2. Regional: Pickup → DC → DC → Pickup
3. National: Pickup → DC → Hub → DC → Pickup
4. Long distance: Pickup → DC → Hub → Hub → DC → Pickup

These are patterns, NOT hardcoded rules.

---

## Routing Strategy

### Hybrid Approach

1. Use rules to reduce search space
2. Apply Dijkstra only when needed
3. Return multiple possible routes

---

## Strategy Pattern

Routing uses Strategy Pattern:

- SameDCStrategy
- SameRegionStrategy
- CrossRegionStrategy

Interface:
```csharp
public interface IRoutingStrategy
{
    List<Route> GetRoutes(Terminal from, Terminal to);
}
```

---

## Dijkstra Usage

Used when:
- Multiple possible paths exist
- Tradeoffs between DC vs Hub

Not used for:
- Simple direct cases

---

## Routing vs Allocation

### Routing Service
- Stateless
- Uses static data
- Returns multiple routes
- Optimizes for time/distance

### Allocation Service
- Uses real-time data
- Handles:
  - Queue load
  - Capacity
- Selects best route

---

## Route Output Design

Instead of primary/secondary:

```json
{
  "routes": [
    { "path": ["A","B","C"], "cost": 10 },
    { "path": ["A","D","C"], "cost": 12 }
  ]
}
```

Allocation decides final route.

---

## Pickup → DC Relationships

- Each Pickup connects to:
  - 1 primary DC
  - 1 secondary DC (optional)

Used for:
- Failover
- Load balancing

---

## Key Design Principles

- Model system as a graph
- Nodes = Terminals
- Edges = Connections
- Use weights for routing decisions
- Avoid hardcoding paths

---

## Exam Key Statement

“Systemet anvender en hybrid routing-strategi, hvor regler reducerer søgeområdet, og Dijkstra anvendes selektivt. Routing returnerer flere mulige ruter, og allocation vælger baseret på real-time kapacitet.”

---

## Recommended Denmark Setup

- 2 Hubs (East/West)
- 5–6 DCs
- 20–50 Pickup Points per DC

---

## Conclusion

- Use Strategy Pattern for routing scenarios
- Use Dijkstra selectively
- Keep routing and allocation separate
- Return multiple routes for flexibility
