# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

GreenGrid is a proof of concept for monitoring about 200 prosumer solar sites: Java 17 / Spring Boot 3.5 microservices on MongoDB and Kafka, secured by Keycloak, delivered by GitOps to a local k3s cluster (Terraform, Argo CD, Argo Rollouts — not yet in the repo). It is a skills showcase built chapter by chapter from a spec; the repo is at the end of Chapter 1 (scaffold: `*Application` classes, empty hexagonal packages, `contextLoads` tests).

## Domain in brief

Customer Success agents register a **Site** (owned by a **Customer**, with an **Installation**: inverter brand, kWp capacity) and onboard it: Registered → ConnectionPending → Verifying → Active (↔ Suspended). **Ingestion** polls each vendor cloud (SolarEdge, Fronius, Enphase — simulated) on the brand's interval, normalizes **Meter Readings** and keeps invalid ones as **Rejected Readings**. **Monitoring** stores readings as time series, derives **Site Health**, opens **Anomalies** (DataGap, Underproduction, RejectionBurst, SourceUnreachable) against **Thresholds** and raises **Alerts**; **Notification** emails them. Roles: Operator (`OPS`), CS agent (`CS`), Customer (`CUSTOMER`).

Read before domain work (on demand, not every session):
- `docs/domain/overview.md` — contexts, context map, site lifecycle, key flows
- `docs/domain/ubiquitous-language.md` — terms, events, words to avoid
- `docs/domain/requirements.md` — FR/NFR, permission matrix, stories with acceptance criteria and rule details, open points
- `docs/research/*.md` — latest research (BR/TC/INC/Q IDs); `docs/plans/2026-09-29-greengrid-roadmap.md` — slice order (S-01…S-18) and where each BR/INC/Q is handled; `docs/plans/*.md` — slice plans; `docs/adr/` — decisions

## Rules that shape the code

- **Ubiquitous language verbatim** in class, event, topic and endpoint names; add new domain words to `docs/domain/ubiquitous-language.md` first.
- **Services integrate only through Kafka events** — no service-to-service REST, no reading another service's database. Each service owns its own database on the shared replica set.
- **State change + event in one Mongo transaction** via the outbox in `platform/greengrid-events`; consumers are idempotent (processed event ids). Valid readings are the exception: Ingestion publishes them directly.
- **Every topic is keyed by site id** (per-site ordering); use reading timestamps, not arrival time.
- **Rejected readings are never dropped** — stored with raw payload and reason.
- **Site Health is derived, never set by hand**; detection applies to Active sites only; each site has an IANA time zone that defines its day and daylight window.
- **Hexagonal layout** under `com.greengrid.<service>` (`site-registry` uses `site_registry`): `domain` (plain Java — no Spring, Mongo, Kafka or Jackson), `application` (use cases, ports), `adapter.in` / `adapter.out`, `config`. Mongo documents are separate classes from domain aggregates. Rules will be enforced by ArchUnit.
- **Java style**: records for value objects (validate in the compact constructor), sealed interfaces for closed sets, Lombok only outside `domain` (no `@Data`/`@Setter`/`@Builder` on aggregates).

## Workflow

Work follows Research → Plan → Implement with the `rresearch`, `pplan` and `iimplement` skills: research writes `docs/research/YYYY-MM-DD-<slug>.md` (+ `.html`), plans go to `docs/plans/` (one per roadmap slice, planned when its dependencies are done), implementation executes an approved plan task by task and logs progress in it. Branches and commits carry story IDs: `feature/REG-01-register-site`, `feat(REG-01): …`, one branch per story or plan slice. Architecture decisions go to `docs/adr/NNN-title.md`.

## Commands

Use the Maven wrapper (no Maven install needed). JDK is pinned in `.sdkmanrc` (`sdk env` → Temurin 17).

```bash
./mvnw clean verify                                   # build + test all modules
./mvnw -pl services/ingestion -am verify              # one module plus what it depends on
./mvnw -pl services/ingestion -am test -Dtest=IngestionApplicationTests -Dsurefire.failIfNoSpecifiedTests=false   # one test class
./mvnw -pl services/ingestion -am test -Dtest='IngestionApplicationTests#contextLoads' -Dsurefire.failIfNoSpecifiedTests=false  # one method
./mvnw -pl services/ingestion spring-boot:run         # run a service
```

`-Dsurefire.failIfNoSpecifiedTests=false` is needed with `-am`, because the other modules in the build don't contain the named test. Once a service depends on `greengrid-events`, run `./mvnw -pl services/<name> -am install -DskipTests` before `spring-boot:run` so the library is in the local repository.

Local infrastructure (Docker must work without `sudo` — Testcontainers-based tests need it too):

```bash
docker compose -f infra/compose/docker-compose.yaml up -d --wait
docker compose -f infra/compose/docker-compose.yaml ps
docker compose -f infra/compose/docker-compose.yaml down -v   # also wipes Mongo data; needed after changing rs.initiate
```

This starts:
- MongoDB 7 replica set `rs0` (mongo1/2/3 on host ports 27017/27018/27019; `mongo-init` runs `rs.initiate` once). Host-run services connect with `directConnection=true` to mongo1. The replica set is required for the outbox's transactions.
- Kafka 4 in KRaft mode — `localhost:9092` from the host, `kafka:19092` inside the compose network.
- Keycloak 26 on `:8080` (admin/admin), importing the `greengrid` realm from `infra/keycloak/greengrid-realm.json` (still empty: roles, client and users come in Chapter 5).
- Mailpit — SMTP on `:1025`, web UI on `:8025` (catches alert emails).

## Architecture

Maven multi-module build; the root `pom.xml` is the parent (Spring Boot 3.5.16 parent + Spring Cloud 2025.0 BOM). Add a module to `<modules>` only once its folder has a POM (`tools/inverter-simulator` is still a placeholder).

- `services/*` — one Spring Boot app per bounded context: `site-registry`, `ingestion`, `monitoring`, `notification`, `gateway`.
- `platform/greengrid-events` — shared library: event envelope and transactional outbox relay (not yet implemented; its Mongo/Kafka dependencies are commented out until then).
- `gateway` — will be Spring Cloud Gateway (reactive) with Keycloak JWT validation; currently scaffolded with the same servlet/Mongo starters as the other services, which must be replaced, not extended.
- `tools/inverter-simulator` — planned simulator for the three vendor APIs.

## Known issues (tracked)

Found by the research and fixed by the approved plan `docs/plans/2026-09-28-registry-foundation-outbox.md` (Phase 1):
- Compose sets `KAFKA_CONTROLLER_LISTENER_NAME` instead of `…_NAMES` (Kafka likely won't start) and mongo-init doesn't give mongo1 `priority: 2`.
- Every service's `application.properties` sets `spring.application.name=site-registry`.
- The README's run instructions are incomplete and use `docker-compose.yml`.
