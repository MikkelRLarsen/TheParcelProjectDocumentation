# Event-drevet Parcel Routing- og Allokeringssystem

Dette projekt implementerer et event-drevet parcel routing- og allokeringssystem inspireret af DAO’s logistikdomæne. Systemet demonstrerer anvendelse af microservices, event-driven arkitektur, choreographed SAGA, CI/CD pipelines og Kubernetes orchestration.

---

## Services og funktioner

| Service | Funktion | Kommunikation | Kommentar |
|---------|---------|--------------|-----------|
| **ParcelService** | Modtager nye pakker via POST `/parcel` endpoint og opretter parcel i systemet. Publicerer `ParcelCreated` event. | Publisher: `ParcelCreated` | Starter parcel-flowet. Kan håndtere cancel-events. |
| **RoutingService** | Beregner optimal route til terminal baseret på shortest-path og kapacitet. Publicerer `RoutingCalculated` event. | Subscriber: `ParcelCreated` <br> Publisher: `RoutingCalculated` | Anvender grafalgoritmer (Dijkstra) til ruteoptimering. |
| **AllocationService** | Allokerer pakker til terminaler, tjekker kapacitet, og placerer pakker i prioriteret kø, hvis terminal er fuld. Publicerer `AllocationDone` event. | Subscriber: `RoutingCalculated` <br> Publisher: `AllocationDone` | Dynamisk kø-strategi baseret på datastruktur og domænelogik. |
| **TerminalService** | Modtager pakker fysisk og bekræfter kapacitet. Returnerer `TerminalConfirmed` eller `TerminalRejected`. | Subscriber: `AllocationRequested` (event eller Dapr Invoke) <br> Publisher: `TerminalConfirmed` / `TerminalRejected` | Kan håndtere synkrone forespørgsler eller event-driven flow. |
| **SAGA Manager** | Holder state for hver parcel (tracker), håndterer cancel-events og udløser kompensations-events til relevante services. | Subscriber: `ParcelCreated`, `RoutingCalculated`, `AllocationDone`, `TerminalConfirmed`, `ParcelCancelled` <br> Publisher: Kompensations-events (`RemoveFromTerminal`, `RemoveFromQueue`, etc.) | Implementerer choreographed SAGA til korrekt kompensation og flow tracking. |

---

## Kommunikation og flow

- **Primært event-driven flow**:  
  `ParcelCreated` → `RoutingCalculated` → `AllocationDone` → `TerminalConfirmed`

- **Kompensation via SAGA Manager**:  
  - `ParcelCancelled` → SAGA Manager læser parcelens state → sender kompensations-events til relevante services  
  - Services reagerer lokalt på kompensations-events og udfører nødvendige handlinger, fx fjern fra kø, fjern fra terminal, log

- **Dapr Pub/Sub**: Alle events håndteres via Dapr, med mulighed for service invocation for read-only forespørgsler (fx Terminal kapacitet).

---

## SAGA Implementering

- **Choreographed SAGA**:  
  - Parcel flowet er event-driven, services reagerer selv på events  
  - SAGA Manager fungerer som **state tracker**, ikke central orkestrator  

- **Kompensation ved cancel-events**:  
  - SAGA Manager sender kompensations-events til services baseret på parcelens position i flowet  
  - Hver service håndterer sin egen kompensation

---

## Algoritmer og datastrukturer

- **RoutingService**:  
  - Grafbaseret terminalnetværk  
  - Shortest-path beregning via Dijkstra-algoritme  

- **AllocationService**:  
  - To kø-strategier: FIFO og prioriteret kø  
  - Dynamisk valg af kø-strategi baseret på pakkens prioritet og terminalkapacitet

---

## CI/CD og deployment

- **Multirepo setup**: Hver service har eget repository og egen CI/CD pipeline via GitHub Actions  
- **Miljøer**: Dev, Test, Prod med miljøspecifik konfiguration  
- **Artefakter**: Immutable og traceable (commit SHA / semver tags)  
- **Dev orchestration**: Aspire til integrationstest  
- **Production orchestration**: Kubernetes til container deployment

---

## Udvidet projektbeskrivelse

Projektet demonstrerer både forståelse for event-driven design, choreographed SAGA til kompensation, multirepo CI/CD pipelines, containeriseret deployment og robust orchestration via Kubernetes. Systemet håndterer parcel-routing effektivt, med dynamisk allokering, shortest-path routing og fleksible kø-strategier, samtidig med at cancel-events håndteres korrekt via SAGA Manager.
