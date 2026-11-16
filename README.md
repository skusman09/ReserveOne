# ReserveOne — Resource Reservation Engine

[![Java](https://img.shields.io/badge/Java-17%20LTS-orange.svg)](https://openjdk.org/projects/jdk/17/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.5.16-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Spring Security](https://img.shields.io/badge/Security-Spring%20Security%206%20%7C%20JWT-blue.svg)](https://spring.io/projects/spring-security)
[![PostgreSQL](https://img.shields.io/badge/Database-PostgreSQL%2016-blue.svg)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Multi--Stage%20Alpine-2496ED.svg)](https://www.docker.com/)
[![Tests](https://img.shields.io/badge/Tests-90%20Passed%20(100%25)-success.svg)]()
[![Coverage](https://img.shields.io/badge/JaCoCo-82%25%20Coverage-green.svg)]()

**ReserveOne** is an enterprise-grade, high-concurrency resource reservation engine. Engineered strictly adhering to the 12-Factor App methodology, it provides zero-collision booking concurrency, optimized multi-table relational data access, and a stateless defense-in-depth security architecture.

Part of the **One** SaaS ecosystem alongside **StaffOne** *(Workforce Management)* and **LibraOne** *(Library Management)*.

---

## 🌟 Key Architectural Highlights

* **Concurrency-Safe Reservation Engine:**
  * Enforces database row-level pessimistic write locking (`LockModeType.PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`), serializing concurrent booking requests at the database layer.
  * Employs time-slot overlap queries using native SQL `EXISTS` short-circuiting, guaranteeing sub-millisecond double-booking prevention under concurrent load.
* **High-Performance Relational Data Access:**
  * Completely eliminates Hibernate $N+1$ query overhead ($O(N) \to O(1)$) using JPA `@EntityGraph(attributePaths = {"resource", "user"})` on multi-table joins.
  * Disables Open Session in View (`spring.jpa.open-in-view: false`) to decouple DB connection lifecycles from HTTP request threads, preventing connection pool exhaustion and memory leaks.
  * Enforces deterministic, proxy-safe entity identity semantics (`equals` and `hashCode`) across all domain models.
* **Authentication, RBAC & Defense-in-Depth Security:**
  * Stateless token-based authentication using **JWT (HMAC-SHA256)** with boot-time key validation enforcing $\ge 256$-bit cryptographic entropy.
  * Strict Role-Based Access Control (`ADMIN` vs. `USER`): Administrators possess full catalog lifecycle permissions, while standard users can only manage their own bookings.
  * Centralized Insecure Direct Object Reference (IDOR) prevention asserting tenant ownership before allowing data mutations.
  * Salted **BCrypt** password hashing ($2^{10}$ key-expansion rounds) with zero password leakage in logs via `@ToString(exclude = "password")`.
* **Dynamic Querying & Specification Engine:**
  * Composable JPA Specifications supporting multi-criteria filtering across booking status, minimum/maximum decimal price ranges, and tenant identity.
  * Safe null-handling returning `cb.conjunction()` (`1=1`) to prevent null pointer exceptions during predicate composition.
  * Spring Data pagination with sensible defaults (`size = 10`, sorted by `createdAt DESC`).
* **Defensive Error Handling & Diagnostics:**
  * RFC 7807-compliant global error translation (`@RestControllerAdvice`) capturing 11 distinct exception categories with structured field-level validation maps.

---

## 🛠 Tech Stack

| Layer | Technology |
| :--- | :--- |
| **Language & Runtime** | Java 17 LTS (Eclipse Temurin) |
| **Core Framework** | Spring Boot 3.5.16 |
| **Security & Auth** | Spring Security 6, JWT (JJWT 0.12.6), BCrypt |
| **Persistence & ORM** | Spring Data JPA, Hibernate ORM 6, HikariCP |
| **Database** | PostgreSQL 16 (Alpine) |
| **API Documentation** | Springdoc OpenAPI 2.8.5, Swagger UI 3.0 |
| **Testing & Quality** | JUnit 5, Mockito 5, TestEntityManager, JaCoCo 0.8.12, H2 Database |
| **DevOps & Containers** | Docker (Multi-Stage Alpine JRE), Docker Compose 3.9 |

---

## 🚀 Quickstart with Docker

The entire platform is fully containerized. To spin up the backend application alongside the PostgreSQL database with automated healthchecks:

```bash
# Clone the repository
git clone https://github.com/skusman09/reserveone-backend.git
cd reserveone-backend

# Build and start services in detached mode
docker-compose up -d --build
```

The REST API will be immediately accessible at: `http://localhost:8080`

To gracefully stop the containers:
```bash
docker-compose down
```

---

## 📖 API Documentation & Exploration

1. **Interactive Swagger UI:**  
   Navigate to `http://localhost:8080/swagger-ui.html` once the container is running. Includes interactive Bearer Token authentication.
2. **OpenAPI 3.0 Specification:**  
   Available as raw JSON at `http://localhost:8080/api-docs`.
3. **Automated Postman Collection:**  
   A pre-configured [`postman_collection.json`](./postman_collection.json) is included in the root directory. It features automated token extraction scripts that dynamically store and inject JWT bearer tokens across requests.

---

## 🔐 Configuration & Environment Variables

All parameters strictly adhere to 12-Factor externalized configuration and can be overridden via environment variables:

| Variable | Default (Local Docker) | Description |
| :--- | :--- | :--- |
| `DATABASE_URL` | `jdbc:postgresql://postgres:5432/resource_booking` | PostgreSQL JDBC connection URL |
| `DATABASE_USER` | `rb_admin` | Database username |
| `DATABASE_PASSWORD` | `RbAdminPassword2026` | Database password |
| `JWT_SECRET` | *(Base64 256-bit key)* | Base64-encoded HMAC-SHA256 signing secret |
| `JWT_EXPIRATION_MS` | `86400000` (24 hours) | Token validity duration in milliseconds |
| `SPRING_PROFILES_ACTIVE` | `dev` | Active Spring profile (`dev`, `prod`, `test`) |
| `JPA_DDL_AUTO` | `validate` (`update` in dev) | Hibernate schema lifecycle strategy |

---

## 🧪 Testing & Code Quality

ReserveOne is validated through a multi-tiered automated test suite, including transactional repository slice tests (`@DataJpaTest`), MockMvc end-to-end security filter chain integration tests, and isolated service unit tests.

```bash
# Run all automated tests and generate JaCoCo coverage reports
./mvnw clean test
```

* **Automated Test Results:** **90 Tests Run, 0 Failures, 0 Errors, 0 Skipped (100% Pass Rate)**
* **JaCoCo Coverage:** **82% Line Coverage** (386 lines covered) with 100% coverage across core security and service classes.
* **Test Report Location:** `target/site/jacoco/index.html`

---

## 👥 Default Seed Accounts (Development & Staging)

When started with the `dev` profile, the application idempotently provisions seed accounts via `DataSeeder`:

| Role | Username | Password | Permissions Scope |
| :--- | :--- | :--- | :--- |
| **Administrator** | `admin` | `admin123` | Full CRUD across resources and all global reservations. |
| **Standard User** | `user1` | `user123` | Read-only resources; create, view, update, cancel own reservations. |

---

## 🏗 Architectural Decisions & Technical Roadmap

**1. Decoupled Financial Pricing Model:**  
Reservation pricing is intentionally encapsulated at the transactional reservation level using high-precision `BigDecimal` (`precision = 10, scale = 2`) to preserve immutable financial audit records even if catalog resource base rates change. A dynamic rate calculation engine (computing resource base rate × duration multiplier) is planned for the next release cycle.

**2. Concurrency Control & Row-Level Locking:**  
To prevent double-booking under high concurrent traffic, the platform utilizes database row-level **Pessimistic Write Locks** (`SELECT ... FOR UPDATE`) coupled with sub-millisecond SQL overlap validation. For globally distributed, multi-region deployments, this will be expanded with distributed locks (e.g., Redis Redlock).

**3. Immutable Audit Trails & Soft Deletion (Phase 2 Roadmap):**  
For regulatory compliance and financial reconciliation, physical entity deletion will transition to a logical soft-delete pattern (`is_active = false`) coupled with Hibernate Envers to maintain a complete historical audit log of reservation lifecycle events.

**4. Distributed Caching via Redis (Phase 2 Roadmap):**  
To scale read-throughput on catalog endpoints (`GET /resources`), a distributed caching layer using **Redis** will be integrated. Caching resource queries and implementing automated cache eviction policies upon resource mutation will drastically reduce database I/O and improve overall API latency.

**5. Edge Rate Limiting & DoS Defense (Phase 2 Roadmap):**  
To protect public authentication routes (`/auth/login`) against brute-force attacks and credential stuffing, token-bucket rate limiting (Bucket4j) will be deployed as a front-facing servlet filter.

**6. Asynchronous Event-Driven Messaging & Load Balancing (Phase 2 Roadmap):**  
To decouple background tasks (such as automated cleanup of expired `PENDING` reservations and confirmation email dispatching) from synchronous HTTP request worker threads, an asynchronous message broker (RabbitMQ / Apache Kafka) will be introduced alongside an Nginx reverse proxy load balancer.
