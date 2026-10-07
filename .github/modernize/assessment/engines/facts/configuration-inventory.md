# Configuration & Externalized Settings Inventory

This project keeps configuration in per-service Spring `application.properties`, Maven POMs, and the frontend's Vite/npm files. It has no environment-specific profiles or external configuration service; database and RabbitMQ are startup dependencies.

## Configuration Sources

| Source | Type | Path/Location | Notes |
|---|---|---|---|
| Maven parent and modules | Build configuration | `pom.xml`, `*/pom.xml` | Nine Spring Boot service modules; versions are declared in each service POM. |
| Spring application configuration | Runtime properties | `*/src/main/resources/application.properties` | Present in all nine services. |
| Maven Wrapper | Build-tool bootstrap | `member-service/`, `catalog-service/`, `borrowing-service/`, `fine-service/`, `notification-service/` | Wrapper distribution points to Maven 3.9.16. The other four services have no wrapper files. |
| Frontend manifest and lockfile | Build/runtime configuration | `frontend/package.json`, `frontend/package-lock.json` | npm lockfile version 3; Vite dev server configured on port 5500. |
| Vite configuration | Frontend runtime | `frontend/vite.config.js` | Dev host `0.0.0.0`, port 5500; preview port 4173. |
| Database initialization script | Database setup | `database/library_db.sql` | Optional manual initialization/sample-data script; services use Hibernate schema update. |
| External config repositories / secret stores | None found | — | No Spring Config Server, Key Vault, Vault, or cloud config integration found. |

## Build Profiles

| Profile | Activation | Purpose | Key Dependencies/Plugins |
|---|---|---|---|
| None defined | — | No Maven build profiles found in the project POMs. | Spring Boot Maven plugin is used by the service modules. |

## Runtime Profiles

| Profile | Activation Method | Config Files | Key Overrides |
|---|---|---|---|
| Default only | Spring default; no `spring.profiles.active` setting found | Each service's `src/main/resources/application.properties` | Service-specific port/name; database and broker settings in the five data/messaging services. |

## Properties Inventory

Common settings in each service with its own configuration:

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.application.name` | Name matching service module | Default | Per-service `application.properties` |
| `server.port` | 8081–8089 (see below) | Default | Per-service `application.properties` |

Database and messaging configuration is present in `member-service`, `catalog-service`, `borrowing-service`, `fine-service`, and `notification-service`:

| Property Key | Default | Profiles | Source |
|---|---|---|---|
| `spring.datasource.url` | `jdbc:postgresql://localhost:5432/library_db` | Default | Per-service `application.properties` |
| `spring.datasource.username` | `postgres` | Default | Per-service `application.properties` |
| `spring.datasource.password` | Member Service contains a literal credential; the other four use `${DB_PASSWORD:changeme}` | Default | Per-service `application.properties`; use an environment variable/secret, never share a real credential |
| `spring.jpa.hibernate.ddl-auto` | `update` | Default | Per-service `application.properties` |
| `spring.jpa.show-sql` | `false` | Default | Per-service `application.properties` |
| `spring.rabbitmq.host` | `localhost` | Default | Per-service `application.properties` |
| `spring.rabbitmq.port` | `5672` | Default | Per-service `application.properties` |
| `spring.rabbitmq.username` | `guest` | Default | Per-service `application.properties` |
| `spring.rabbitmq.password` | Configured in each properties file; do not distribute real credentials | Default | Per-service `application.properties` |

Additional settings:

| Service | Property Key | Default | Source |
|---|---|---|---|
| Borrowing | `member-service.url` | `http://localhost:8081` | `borrowing-service/src/main/resources/application.properties` |
| Borrowing | `catalog-service.url` | `http://localhost:8082` | `borrowing-service/src/main/resources/application.properties` |
| Borrowing | `library.borrowing.loan-period-days` | `14` | `borrowing-service/src/main/resources/application.properties` |
| Borrowing | `library.overdue-check.cron` | `0 0 * * * *` | `borrowing-service/src/main/resources/application.properties` |
| Fine | `library.fine.per-day-amount` | `10.00` | `fine-service/src/main/resources/application.properties` |
| Notification | `library.notification.retry.max-attempts` | `3` | `notification-service/src/main/resources/application.properties` |
| Notification | `library.notification.retry.backoff-ms` | `2000` | `notification-service/src/main/resources/application.properties` |

| Service | Port |
|---|---:|
| Member | 8081 |
| Catalog | 8082 |
| Borrowing | 8083 |
| Fine | 8084 |
| Notification | 8085 |
| Authentication | 8086 |
| API Gateway | 8087 |
| Audit | 8088 |
| Recommendation | 8089 |

Authentication, API Gateway, Audit, and Recommendation currently have only their service name and port in `application.properties`.

## Startup Parameters & Resource Requirements

| Service | JVM/Runtime Options | Memory | Instance Count |
|---|---|---|---|
| All nine Spring Boot services | No JVM flags or heap settings found | Not specified | Not specified |
| Frontend | Vite dev server; `npm run dev` | Not specified | One local dev server |

## Startup Dependency Chain

1. Start PostgreSQL and create database `library_db` (port 5432).
2. Start RabbitMQ (AMQP port 5672; Management UI port 15672 is optional for application operation).
3. Start Member (8081) and Catalog (8082) before Borrowing (8083), which calls both services synchronously.
4. Start Fine, Notification, Authentication, API Gateway, Audit, and Recommendation as needed. The README lists ports 8084–8089.
5. Start the frontend with `npm run dev` (port 5500).

No readiness probes, container orchestration, or automated wait-for dependency mechanism was found. The README instructs starting dependencies/services manually.

## Secrets & Sensitive Configuration

| Secret Reference | Type | Storage (masked) |
|---|---|---|
| PostgreSQL password used by Member Service | Database credential | Literal value currently present in `member-service/src/main/resources/application.properties`; do not copy/share it. Rotate before sharing and replace with a secret/environment-variable reference. |
| PostgreSQL password used by Catalog, Borrowing, Fine, and Notification | Database credential | `${DB_PASSWORD:changeme}` placeholder in the service properties. Configure the actual value securely in the environment. |
| RabbitMQ password | Broker credential | Configured in the five service properties; treat as sensitive and replace defaults for shared/non-local environments. |

### Secrets Provisioning Workflow

For local development, set database credentials in the environment before starting the relevant services. The README demonstrates a terminal-scoped PowerShell environment variable. Member Service needs special attention because its properties file currently contains a literal password. Before sharing this project or pushing it to a shared repository, rotate that credential and externalize it. No managed identity or secret-store integration was found.

## Feature Flags

| Flag Name | Default | Controlled By |
|---|---|---|
| None found | — | No feature-flag framework or conditional feature toggles identified. |

## Framework & Runtime Versions

| Component | Version | Source |
|---|---|---|
| Java | 21 | `java.version` in all nine service POMs |
| Spring Boot | 4.1.1 | Parent version in all nine service POMs |
| Maven | 3.9.16 via wrapper | Maven Wrapper distribution URL in the five services that include wrapper files; wrapper is absent from Authentication, API Gateway, Audit, and Recommendation |
| PostgreSQL | 18 (README's development target) | README; JDBC URL uses local default port 5432 |
| RabbitMQ | Not pinned in project | README specifies RabbitMQ with Management plugin and AMQP port 5672; choose RabbitMQ and Erlang versions from the official compatibility matrix |
| React / React DOM | `^18.2.0` | `frontend/package.json` |
| Vite | `^5.4.10` | `frontend/package.json` |
| `@vitejs/plugin-react` | `^4.3.2` | `frontend/package.json` |
| Node.js / npm | Not pinned in project | Required to install and run the frontend; use a compatible Node.js LTS release |
| Postman | Not pinned in project | Optional API testing tool; collection is under `postman/` |
