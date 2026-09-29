# Ubiquitous language

> These terms are used **verbatim** in class names, event names, topics and endpoints, and each means exactly one thing inside its context. Source: spec Part I §1.5, plus the additions marked *(added)* from the research of 2026-09-27. When code, docs or conversation need a new domain word, add it here first.

## Terms

| Term | Meaning | Context | Code shape |
|---|---|---|---|
| Site | Physical location with solar equipment, owned by one Customer | Registry (source of truth), replicated elsewhere | aggregate `Site`, id `SiteId` (`site-` + UUID) |
| Customer | The site owner; in this PoC a reference (id + name) only | Registry | value object `CustomerRef` |
| Region | Area code of a site (e.g. `DE-BY`); used for filtering and to derive the default time zone *(added)* | Registry, Monitoring | value object `Region` |
| Site time zone | IANA time zone of a site (e.g. `Europe/Berlin`); defines its local day and daylight window *(added, Q-011)* | Registry, Monitoring | `ZoneId` on the site and in site events |
| Installation | The equipment at a site: inverter brand, capacity (kWp), optional battery (kWh), smart meter | Registry | value object `Installation` |
| Inverter Brand | SolarEdge, Fronius or Enphase; decides adapter and polling interval | Registry, Ingestion | enum `InverterBrand` |
| Onboarding Status | Registered, ConnectionPending, Verifying, Active, Suspended | Registry | enum `OnboardingStatus` |
| Connection | How GreenGrid reaches a site's vendor cloud: brand + external site id + credential reference (never the secret itself) | Registry, Ingestion | value object `Connection` |
| Reading Source | Ingestion's local view of a site it must poll (brand, interval, mode verifying/normal, next poll, status) | Ingestion | aggregate `ReadingSource` |
| Raw Reading | A vendor payload as received, before normalization | Ingestion | vendor DTOs stay inside their adapter |
| Meter Reading | Normalized reading: site, timestamp (UTC), produced kWh, consumed kWh (optional), interval | Ingestion → Monitoring | event `MeterReadingRecorded` |
| Rejected Reading | A raw reading that failed validation, kept with reason and raw payload for audit and review | Ingestion | aggregate `RejectedReading`, event `ReadingRejected` |
| Rejection Reason | FutureTimestamp, NegativeValue, CounterReset, DuplicateTimestamp, MalformedPayload | Ingestion | enum `RejectionReason` |
| Resubmission | An operator's decision that a rejected reading is valid and must be reprocessed | Ingestion | — |
| Poll Failure | Vendor API unreachable or erroring for a site after retries | Ingestion | event `ReadingSourceFailed` |
| Site Health | Healthy, Degraded, Offline, Unknown — derived, never set by hand | Monitoring | enum `SiteHealth` |
| Threshold | A configurable limit (max gap factor, min production ratio, rejection burst), default or per site | Monitoring | aggregate `Thresholds` |
| Baseline | Expected production for a site and hour, derived from capacity (PoC: simple clear-sky model) | Monitoring | — |
| Anomaly | A detected problem with a lifecycle Open → Acknowledged → Resolved | Monitoring | aggregate `Anomaly` |
| Anomaly Type | DataGap, Underproduction, RejectionBurst, SourceUnreachable | Monitoring | enum `AnomalyType` |
| Alert | Notification-worthy fact that an anomaly was opened | Monitoring → Notification | event `AlertTriggered` |
| Operator | Operations user (Diana's role); realm role `OPS` | Identity | — |
| CS Agent | Customer Success user (Tomás, Maria); realm role `CS` | Identity | — |
| Customer (role) | Site owner logging in; realm role `CUSTOMER`, token claim `customerId` | Identity | — |

## Events

| Event | Producer | Topic | Meaning |
|---|---|---|---|
| SiteRegistered | site-registry | `greengrid.registry.site.v1` | A site was registered (status Registered) |
| SiteConnectionConfigured | site-registry | `greengrid.registry.site.v1` | Connection details added or changed (status ConnectionPending) |
| SiteActivated | site-registry | `greengrid.registry.site.v1` | Verification succeeded (status Active) |
| SiteSuspended | site-registry | `greengrid.registry.site.v1` | A CS agent suspended the site, with a reason |
| SiteReactivated | site-registry | `greengrid.registry.site.v1` | A suspended site became Active again |
| MeterReadingRecorded | ingestion | `greengrid.ingestion.reading.v1` | A valid, normalized reading |
| ReadingRejected | ingestion | `greengrid.ingestion.reading.v1` | A reading failed validation (reason attached) |
| ReadingSourceVerified | ingestion | `greengrid.ingestion.source.v1` | Successful poll(s) during verification, with running count |
| ReadingSourceFailed | ingestion | `greengrid.ingestion.source.v1` | Polling failed after retries |
| AlertTriggered | monitoring | `greengrid.monitoring.alert.v1` | An anomaly opened |
| AnomalyResolved | monitoring | `greengrid.monitoring.alert.v1` | An anomaly was resolved |

All events share one envelope — `eventId` (drives idempotency), `eventType`, `eventVersion` (drives schema evolution), `occurredAt`, `aggregateId`, `producer`, `payload` — and every topic is keyed by site id. Each topic has a `.DLT` twin for poison messages.

## Words to avoid

| Avoid | Use | Why |
|---|---|---|
| plant, system, installation (for the site) | Site | Installation is the equipment, not the place |
| location (for the site's area) | Region | The model and filters use Region |
| status (for health) | Site Health | Onboarding Status and Site Health are different things |
| invalid / failed reading | Rejected Reading | "Failed" is used for Poll Failure |
| notification (for the domain fact) | Alert | Notification is the delivery context |
| RawReading for normalized data | Meter Reading (or a named adapter-internal type) | Raw Reading means "before normalization" (research INC-024) |
