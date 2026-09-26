# GreenGrid Java Edition

A proof of concept for monitoring renewable energy sites, built as Java 17 / Spring Boot 3.5 microservices on MongoDB and Kafka. Sites are registered and onboarded, inverters are polled, readings are validated and stored as time series, and anomalies (data gaps, underproduction, rejection bursts) raise email alerts. The system deploys to a local Kubernetes cluster through Terraform, Argo CD and Argo Rollouts.

## Stack

| Area | Choice |
| --- | --- |
| Language / framework | Java 17, Spring Boot 3|
| Data | MongoDB 7 (3-node replica set, one database per service) |
| Messaging | Apache Kafka (KRaft), transactional outbox |
| Security | Keycloak 26, OAuth2 / JWT, Spring Cloud Gateway |
| Delivery | GitHub Actions, GHCR, Terraform, Argo CD, Argo Rollouts |
| Architecture | Bounded contexts, hexagonal layout, ArchUnit rules |

## Prerequisites

- Temurin JDK 17 (for example through SDKMAN)
- Docker with Compose v2. On Windows, use Docker Desktop with the WSL 2 engine and WSL integration turned on, and keep the repo inside the WSL filesystem.
- Your user in the `docker` group (`sudo usermod -aG docker $USER`, then restart WSL)

Maven doesn't need to be installed; the project uses the Maven wrapper.

## Getting started

```bash
# 1. Start local infrastructure
docker compose -f infra/compose/docker-compose.yml up -d
docker compose -f infra/compose/docker-compose.yml ps   # wait until everything is up