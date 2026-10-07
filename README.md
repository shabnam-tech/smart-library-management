# Smart Library Management System

A library circulation application built as a collection of Spring Boot microservices, with a React web interface. The services manage members, books, borrowings, fines, notifications, authentication, audit events, and recommendations. PostgreSQL stores application data, and RabbitMQ carries events between services.

> **Status:** This is a local-development/demo project. The services and UI are intended to run on one machine. Review the security and production notes before deploying or using real member data.

## Contents

- [Features](#features)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Run locally](#run-locally)
- [Build and test](#build-and-test)
- [Service endpoints](#service-endpoints)
- [Postman collection](#postman-collection)
- [Project structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Security and production notes](#security-and-production-notes)

## Features

- Register and view library members.
- Add and browse books and available copy counts.
- Borrow and return books, with member and book checks before a borrowing is created.
- Calculate and pay overdue fines.
- Create and view notifications from library events.
- Publish and consume library events asynchronously through RabbitMQ.
- View audit events and request popular-book or member recommendations through their service APIs.
- Use the React dashboard for the main member, catalog, borrowing, fine, and notification workflows.

## Technology stack

- **Backend:** Java 21, Spring Boot 4.1.1, Maven
- **Frontend:** React 18, Vite 5
- **Database:** PostgreSQL
- **Messaging:** RabbitMQ
- **API testing:** Postman

## Architecture

The backend consists of nine independently runnable Spring Boot services:

| Service | Responsibility | Port |
| --- | --- | ---: |
| Member Service | Member records | 8081 |
| Catalog Service | Books and available copies | 8082 |
| Borrowing Service | Borrowing and returns; checks member and book services | 8083 |
| Fine Service | Fine records and payment | 8084 |
| Notification Service | Notifications generated from events | 8085 |
| Authentication Service | Login and token validation endpoints | 8086 |
| API Gateway Service | Gateway information and health endpoints | 8087 |
| Audit Service | Audit event API | 8088 |
| Recommendation Service | Popular-book and member recommendation endpoints | 8089 |

Member, Catalog, Borrowing, Fine, and Notification Services use the local `library_db` PostgreSQL database. RabbitMQ is used by the event-driven services. Borrowing Service makes synchronous HTTP calls to Member and Catalog before accepting a borrowing request. The React frontend currently calls the Member, Catalog, Borrowing, Fine, and Notification services directly; the other service APIs can be accessed on their listed ports.

The frontend is served by Vite at `http://localhost:5500`. PostgreSQL uses its default port `5432`; RabbitMQ uses `5672` for AMQP and `15672` for its optional management UI.

### Event flow

Services publish library events through the RabbitMQ topic exchange `library.events`. Relevant routing keys include:

| Event | Published by | Used by |
| --- | --- | --- |
| `book.borrowed` | Borrowing Service | Catalog and Notification Services |
| `book.returned` | Borrowing Service | Catalog, Fine, and Notification Services |
| `book.overdue` | Borrowing Service | Fine Service |
| `fine.generated`, `fine.paid` | Fine Service | Notification Service |
| `book.available` | Catalog Service | Notification Service |

Notification Service configures bounded retries and dead-letter queues for its event consumers. Consumers use event IDs to avoid applying duplicate deliveries more than once. This project does not implement a transactional outbox, so it should not be treated as a production-grade guarantee of exactly-once event processing.

## Prerequisites

Install and start the following before running the project:

- Java Development Kit (JDK) 21
- Maven 3.9 or later
- Node.js and npm
- PostgreSQL
- RabbitMQ and Erlang/OTP (use versions supported by your RabbitMQ release)

The Maven Wrapper is included in some service folders, but not at the repository root or in every service. The commands below use an installed `mvn` executable from the repository root.

## Configuration

The default local configuration expects:

- PostgreSQL at `localhost:5432`
- Database name `library_db`
- PostgreSQL user `postgres`
- RabbitMQ at `localhost:5672`, using the local `guest` account

Create the database once if it does not already exist:

```powershell
createdb -U postgres library_db
```

Alternatively, run this from `psql` while connected as a database administrator:

```sql
CREATE DATABASE library_db;
```

The services create or update their tables when they start. The optional schema script is in [`database/library_db.sql`](database/library_db.sql).

Catalog, Borrowing, Fine, and Notification Services are configured to read the database password from `SPRING_DATASOURCE_PASSWORD`. In PowerShell, set it separately in each service terminal:

```powershell
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
```

Member Service's datasource setting should also use an environment-variable placeholder such as `spring.datasource.password=${SPRING_DATASOURCE_PASSWORD}`. Check its `application.properties` and replace any literal local password before relying on the environment variable or sharing this repository. Do not put real credentials in this README, source control, screenshots, or shared terminal logs. If a credential has already been committed, rotate it; removing it from the current file does not remove it from Git history.

RabbitMQ's management page, when the management plugin is enabled, is at `http://localhost:15672`. The default `guest` account is intended only for local use.

## Run locally

### 1. Start PostgreSQL and RabbitMQ

Make sure both services are running, and that the `library_db` database exists. Keep their default local ports available (`5432`, `5672`).

### 2. Start the backend services

Open a separate terminal for each service. In each terminal, change to the repository root, set the PostgreSQL password for services that use the database, then run that service's command. Keep the terminals open while using the application.

Start Member and Catalog before Borrowing because Borrowing calls them synchronously. For example, in the first terminal:

```powershell
cd C:\path\to\smart-library-management
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
mvn -pl member-service spring-boot:run
```

Wait until the log reports that the application has started. Repeat in separate terminals for the remaining services:

```powershell
cd C:\path\to\smart-library-management
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
mvn -pl catalog-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
mvn -pl borrowing-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
mvn -pl fine-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
$env:SPRING_DATASOURCE_PASSWORD = Read-Host "PostgreSQL password"
mvn -pl notification-service spring-boot:run
```

Authentication, API Gateway, Audit, and Recommendation do not use the shared application database in their current configuration:

```powershell
cd C:\path\to\smart-library-management
mvn -pl authentication-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
mvn -pl api-gateway-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
mvn -pl audit-service spring-boot:run
```

```powershell
cd C:\path\to\smart-library-management
mvn -pl recommendation-service spring-boot:run
```

For this Windows-specific setup, replace `C:\path\to\smart-library-management` with the path where you cloned the project. If a service fails during startup, resolve that error before starting services that depend on it.

### 3. Start the React frontend

Open another terminal:

```powershell
cd C:\path\to\smart-library-management\frontend
npm ci
npm run dev
```

Open **http://localhost:5500** in your browser. Leave the Vite terminal and backend terminals running. Stop a service with `Ctrl+C` in its terminal when you are finished.

## Build and test

From the repository root, build all Maven modules:

```powershell
mvn clean package
```

Run the frontend production build:

```powershell
cd frontend
npm ci
npm run build
```

To run a service's Maven tests, use:

```powershell
mvn -pl member-service test
```

Replace `member-service` with the module you want to test. Start the frontend's local development server with `npm run dev`; use `npm run preview` to preview a production build locally.

## Service endpoints

The primary REST resources are:

| Service | Endpoint examples |
| --- | --- |
| Member | `GET/POST /members`, `GET /members/{id}` |
| Catalog | `GET/POST /books`, `GET /books/{id}` |
| Borrowing | `GET/POST /borrowings`, `GET /borrowings/{id}`, `PUT /borrowings/{id}/return` |
| Fine | `GET /fines`, `GET /members/{id}/fines`, `PUT /fines/{id}/pay` |
| Notification | `GET /notifications`, `GET /notifications/member/{memberId}` |
| Authentication | `POST /auth/login`, `GET /auth/validate` |
| API Gateway | `GET /gateway/health`, `GET /gateway/services`, `GET /gateway/route/{serviceName}` |
| Audit | `GET/POST /audit/events` |
| Recommendation | `GET /recommendations/popular`, `GET /recommendations/member/{memberId}` |

For example, the Member Service API is available at `http://localhost:8081/members` when that service is running.

## Postman collection

Import [`postman/Smart-Library.postman_collection.json`](postman/Smart-Library.postman_collection.json) into Postman. Start the relevant services first, then use the collection requests to exercise the REST APIs.

## Project structure

```text
smart-library-management/
├── api-gateway-service/
├── audit-service/
├── authentication-service/
├── borrowing-service/
├── catalog-service/
├── fine-service/
├── member-service/
├── notification-service/
├── recommendation-service/
├── frontend/                 # React + Vite application
├── database/                  # Optional SQL schema / seed script
├── postman/                   # Postman collection
├── pom.xml                    # Maven reactor / module list
└── README.md
```

## Troubleshooting

- **Database authentication failed:** verify the PostgreSQL password and ensure the service has the correct password configuration. Environment variables set in one terminal do not automatically apply to other terminals.
- **Connection refused on port 5432:** start PostgreSQL and verify its configured port.
- **Connection refused on port 5672:** start RabbitMQ and verify its AMQP listener. The management UI on `15672` is optional and is not the AMQP listener.
- **Port already in use:** stop the process using the service's port or change that service's `server.port`.
- **Borrowing cannot reach Member or Catalog:** start Member Service on `8081` and Catalog Service on `8082` before Borrowing Service.
- **Catalog reports stale classes or unresolved compilation problems:** from the repository root, run `mvn -pl catalog-service clean spring-boot:run`.
- **Frontend cannot reach a backend:** confirm that the corresponding service is running on its configured port and check the browser developer console for the failing request.
- **Maven fails to resolve dependencies:** check network access and try again; the first build downloads dependencies from Maven repositories.

## Security and production notes

- This project is configured for local development, not production deployment.
- Do not commit database passwords, RabbitMQ credentials, signing keys, or other secrets. If a credential has been committed, remove it from configuration and rotate it; deleting it from the latest version does not remove it from Git history.
- Authentication and API Gateway services exist, but the current React application calls the application services directly. Do not assume the UI is protected by the authentication service.
- Review authorization, input validation, CORS, TLS, secrets management, logging of personal data, database access, and RabbitMQ permissions before exposing any service to a network.
- Back up the database before changing schemas or testing destructive operations.
