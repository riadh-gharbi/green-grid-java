# GreenGrid — requirements

> Adapted from spec Part I §2–§3 and the rule details of Part II (Sep 24, 2026). Changes decided in the research clarifications of 2026-09-27 are marked **(decided: Q-00x)**; see `docs/research/2026-09-27-greengrid-spec-baseline.md` §8. Research rule IDs (BR-###) are given where the research tracks the story. Story IDs are used in branch names (`feature/REG-01-register-site`) and commit messages (`feat(REG-01): …`).

## Contents
1. Functional requirements
2. Non-functional requirements
3. Roles and permissions
4. Epics and stories (with rule details)
5. Open points

## 1. Functional requirements

| ID | Requirement | Epic |
|---|---|---|
| FR-1 | CS agents register sites, add connection details and move a site through onboarding | REG |
| FR-2 | The system polls each Active or Verifying site on its brand's interval and normalizes readings | ING |
| FR-3 | Invalid readings are stored with a reason, never dropped; operators can resubmit or discard them | ING |
| FR-4 | Readings are stored per site as time series and summarized per day (the site's local day — **decided: Q-011**) | MON |
| FR-5 | Site health is derived continuously from data gaps, production vs baseline, rejections and poll failures | MON |
| FR-6 | Operators configure default and per-site thresholds; Monitoring owns them | MON |
| FR-7 | Anomalies have a lifecycle (open, acknowledge, resolve) and each new anomaly raises an alert | MON |
| FR-8 | Alerts are delivered by email (local mail catcher) | NOT |
| FR-9 | Customers see only their own sites' health and daily production | MON, AUTH |
| FR-10 | All user access goes through one gateway with OAuth2 tokens from Keycloak | AUTH |

## 2. Non-functional requirements

| ID | Quality | Target (PoC) |
|---|---|---|
| NFR-1 | Detection latency | A data gap is detected within the site's max gap (default 2 × its interval): ≤ 15 min for 5-minute sites, longer for slower brands (**decided: Q-001** — replaces "≤ 15 min for every site"; research BR-039) |
| NFR-2 | Throughput | 200 sites at 5-min intervals (~58k readings/day) with headroom to 10× in a load test |
| NFR-3 | Query performance | Site health list < 100 ms; one site's day of readings < 50 ms at p95, locally |
| NFR-4 | Reliability | No lost events between database writes and Kafka (transactional outbox); idempotent consumers |
| NFR-5 | Auditability | Rejected readings retained with raw payload, reason and review history |
| NFR-6 | Security | JWT validated at the gateway and in every service; least-privilege roles; no secrets in Git |
| NFR-7 | Deployability | Git push to running canary with no manual kubectl |
| NFR-8 | Safe releases | Canary with automated metric analysis and automatic rollback |
| NFR-9 | Evolvability | Versioned events and documents; old versions still readable |
| NFR-10 | Architecture fitness | Layer and dependency rules enforced by ArchUnit in CI |

## 3. Roles and permissions

| Capability | Operator (OPS) | CS agent (CS) | Customer (CUSTOMER) |
|---|---|---|---|
| Register site, edit installation and connection | — | Yes | — |
| Move site through onboarding, suspend, reactivate | — | Yes | — |
| View onboarding pipeline | Read | Yes | — |
| View all sites' health and anomalies | Yes | Read | — |
| Acknowledge and resolve anomalies (manual resolve with a note — **decided: Q-002**) | Yes | — | — |
| Configure thresholds | Yes | — | — |
| Review, resubmit or discard rejected readings | Yes | — | — |
| View own sites' health and daily production | — | — | Yes (ownership enforced; other sites → 404) |

Denied capabilities return 403, except a customer asking for another customer's site, which returns 404 (existence is not revealed). Missing or expired token → 401, at the gateway and on direct service calls.

## 4. Epics and stories

Acceptance criteria are Given/When/Then; each becomes at least one named test. Track: Core unless marked *Stretch*.

### PLAT — Platform and delivery

| Story | Acceptance (short) | BR |
|---|---|---|
| PLAT-01 Monorepo with parent POM | Fresh clone: `./mvnw verify` builds all modules, tests pass | BR-001 |
| PLAT-02 Local infrastructure in one command | `docker compose up` gives a healthy 3-node Mongo replica set, Kafka (KRaft), Keycloak, Mailpit | BR-002 |
| PLAT-03 Event envelope and outbox | Event written to the outbox in the same transaction as the aggregate; published to Kafka at least once | BR-003 |
| PLAT-04 Architecture rules | A domain class importing Spring/Mongo types fails the build (ArchUnit) | BR-004 |
| PLAT-05 Container images | Image < 250 MB, non-root, health probes | BR-005 |
| PLAT-06 Platform by Terraform | `terraform apply` on the home-server k3s installs Argo CD, Rollouts, Strimzi, Prometheus; infra app synced | BR-006 |
| PLAT-07 CI publishes images | Push to main: tests pass, images to GHCR tagged with the SHA, GitOps repo updated | BR-007 |
| PLAT-08 Canary releases | 20% → 50% → 100% with analysis; automatic rollback above 5% errors | BR-008 |
| PLAT-09 AWS modules (*Stretch*) | `terraform validate` and `tflint` pass without credentials | BR-009 |

### REG — Site Registry and onboarding

| Story | Acceptance (short) | BR |
|---|---|---|
| REG-01 Register a site | Valid customer, region and installation → Registered, `SiteRegistered` published; capacity ≤ 0 kWp → domain error | BR-010 |
| REG-02 Add connection details | Registered site + brand-specific connection → ConnectionPending, `SiteConnectionConfigured` published | BR-011 |
| REG-03 Verification moves sites forward | First successful poll → Verifying; 3 successful polls → Active, `SiteActivated` published | BR-012 |
| REG-04 Suspend and reactivate | Active + reason → Suspended, `SiteSuspended`, polling stops; Suspended → Active, `SiteReactivated`, polling resumes (**decided: Q-003**) | BR-013 |
| REG-05 Onboarding pipeline | Counts per status and sites stuck > 48 h in ConnectionPending | BR-014 |
| REG-06 Audit trail (*Stretch*) | Every transition shows who changed what and when | BR-015 |

Rule details:
- Site id = `site-` + UUID; timestamps truncated to milliseconds. Capacity must be > 0 kWp, battery ≥ 0; brand, customer id and region are required.
- The site's time zone is stored at registration (derived from region unless given) and carried in site events (**decided: Q-011**).
- Connection can be (re)configured in Registered or ConnectionPending; its brand must equal the installation brand; the credential is a reference and is never returned.
- Verification: the count of successful polls is the maximum of stored and reported; at 3 (configurable) the site becomes Active. A failure while Verifying returns the site to ConnectionPending with count 0; failures while ConnectionPending are ignored; reports for Active/Suspended sites are ignored; duplicate reports have no effect.
- Suspend only from Active with a non-blank reason; reactivate only from Suspended.
- HTTP mapping: shape errors 400, domain rule violations 422, invalid transitions 409 (naming the current status and action), not found 404, concurrent update 409.

### ING — Data Ingestion

| Story | Acceptance (short) | BR |
|---|---|---|
| ING-01 Local list of sources | `SiteConnectionConfigured`/`SiteActivated` create or update a Reading Source; `SiteSuspended` stops polling; `SiteReactivated` resumes normal polling (**decided: Q-003**) | BR-016 |
| ING-02 Poll on the brand interval | Exactly one poll per interval per source, even with 2 Ingestion replicas | BR-017 |
| ING-03 One adapter per brand | SolarEdge, Fronius or Enphase payload → normalized Meter Reading in UTC and kWh | BR-018 |
| ING-04 Invalid readings kept | Future timestamp, negative value, counter reset, duplicate or malformed payload → Rejected Reading with reason and raw payload, `ReadingRejected` published | BR-019 |
| ING-05 Valid readings as events | Valid reading → `MeterReadingRecorded` keyed by site id | BR-020 |
| ING-06 Poll failures surfaced | Vendor error/timeout, retries exhausted → `ReadingSourceFailed` | BR-021 |
| ING-07 Resubmit or discard (*Stretch*) | Resubmit with a note → reprocessed bypassing the failed rule, marked Resubmitted | BR-022 |

Rule details:
- A timestamp more than 5 minutes in the future is FutureTimestamp; rules run in the order listed in ING-04.
- Vendor calls: timeout, 3 attempts with exponential backoff; then `ReadingSourceFailed` and the next poll is postponed.
- In verification mode Ingestion reports `ReadingSourceVerified` with a running success count; on `SiteActivated` the source switches to normal mode.
- Brand intervals: Enphase-style sources report every 60 minutes; SolarEdge and Fronius use 15 and 5 minutes (assignment open — see §5).

### MON — Site Health Monitoring

| Story | Acceptance (short) | BR |
|---|---|---|
| MON-01 Readings as time series | Each `MeterReadingRecorded` stored once, even if delivered twice | BR-023 |
| MON-02 Site health overview | Status, last reading time and open anomaly count per site, 200 sites in < 100 ms | BR-024 |
| MON-03 Data gaps detected | No reading within the max gap (default 2 × interval) for an Active site → DataGap anomaly, site Offline | BR-025 |
| MON-04 Underproduction detected | Below the min ratio (default 40%) of the baseline for 3 consecutive intervals in daylight → Underproduction anomaly | BR-026 |
| MON-05 Rejection bursts detected | 5 consecutive rejections or > 20% in 1 h for a site → one RejectionBurst anomaly | BR-027 |
| MON-06 Configurable thresholds | Per-site override used instead of the default; ratio > 1 (or gap factor < 1) rejected | BR-028 |
| MON-07 Anomaly lifecycle and alerts | Open → Acknowledged by an operator; resolved manually by an operator (**decided: Q-002**) or automatically when the condition clears; each opening publishes `AlertTriggered` exactly once | BR-029 |
| MON-08 Customer view (*Stretch*) | Customer owning sites A and B sees A and B; site C → 404 | BR-030 |
| MON-09 Unreachable sources (*Stretch*) | `ReadingSourceFailed` → SourceUnreachable anomaly | BR-031 |

Rule details:
- Detection applies to **Active sites only**; suspending a site resolves its open anomalies and shows its health as Unknown; activation and reactivation start the gap clock, so a site that never reports is flagged too (**decided: Q-012**).
- Daylight and "per day" are evaluated in the site's time zone (**decided: Q-011**); the spec's example window is 08:00–18:00.
- Baseline (PoC): expected kWh per interval = capacity × clear-sky factor for the hour × interval length.
- At most one active anomaly per site and type; late readings never move "last reading" backwards.
- Threshold defaults: max gap factor 2 (≥ 1), min production ratio 0.4 (0–1), rejection burst 5 consecutive or 20% per hour.

### NOT — Alert Notification

| Story | Acceptance (short) | BR |
|---|---|---|
| NOT-01 Alerts by email | One email per `AlertTriggered` with site, anomaly type and time; duplicates ignored | BR-032 |
| NOT-02 No alert storms (*Stretch*) | 10 alerts for one site within 5 min → one grouped email | BR-033 |

### AUTH — Authentication and authorization

| Story | Acceptance (short) | BR |
|---|---|---|
| AUTH-01 Log in and call the API | Demo user completes authorization code + PKCE (in Bruno) → JWT with realm roles | BR-034 |
| AUTH-02 Every request authenticated | No or expired token → 401 at the gateway and on direct service calls | BR-035 |
| AUTH-03 Permission matrix enforced | CS resolving an anomaly → 403; customer reading another customer's site → 404; every row of §3 tested | BR-036 |

Demo users: 1 operator, 2 CS agents, 2 customers; public client `greengrid-api`; `customerId` user attribute mapped into the token; no password grant.

### Without a story ID

- **Inverter simulator** (BR-037): 200 deterministic sites, three vendor payload styles with their own intervals, daylight × capacity × weather production, fault injection (offline, future timestamps, counter reset, underproduce, HTTP 500), protected by a static API key.
- **Gateway resilience** (BR-038): one port for all routes, correlation id on every request, per-route timeouts, a clean 503 problem response when a service is down; rate limiting is *Stretch*.

## 5. Open points

Deferrable questions from the research (none blocks current work; decide them when planning the slice named):

| Research Q | Question | Decide in |
|---|---|---|
| Q-004 | Which interval do SolarEdge and Fronius use (15 / 5 min)? | Ingestion / simulator plan |
| Q-006 | What makes a site Degraded? | Monitoring plan |
| Q-007 | Per-service database credentials in the PoC? | Kubernetes plan |
| Q-008 | Keep `AnomalyOpened` and "escalated" alerts? | Monitoring plan |
| Q-013 | What clears RejectionBurst and SourceUnreachable automatically? | Monitoring plan |
| Q-014 | Does Ingestion reset its verification count after a failure or reconfiguration? | REG-03 plan |
| Q-015 | Kafka retention for the readings topic (system of record)? | Readings plan |
