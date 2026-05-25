---
name: mindata-sling hotel-search-service
description: Backend engineer test — Spring Boot 3.3 + Java 21 microservice with Kafka, PostgreSQL, hexagonal architecture
type: project
---

Full Spring Boot 3.3.6 / Java 21 microservice built from scratch.

**Why:** Mindata/Sling backend engineer technical assessment.

**Key design decisions:**
- Hexagonal architecture (ports & adapters), no Maven modules
- Domain: `HotelSearch` record (immutable), `SearchNotFoundException`
- Input ports: `SearchUseCase`, `CountUseCase`, `PersistSearchUseCase`
- Output ports: `SearchPersistencePort`, `SearchMessagePort`
- REST inbound adapter: `SearchController` (POST /search, GET /count)
- Kafka inbound adapter: `SearchKafkaConsumer` (persists from topic)
- Kafka outbound adapter: `SearchKafkaProducer`
- Persistence outbound adapter: `SearchPersistenceAdapter` + `SearchJpaRepository`
- Ages stored as JSON TEXT in DB; native SQL count query preserves order semantics
- Virtual threads: Tomcat via `TomcatProtocolHandlerCustomizer`, Kafka consumer via `SimpleAsyncTaskExecutor(virtualThreads=true)`
- Flyway for DB migrations; H2 in-memory for tests (`@DataJpaTest`)
- JaCoCo 80% line/branch/method threshold enforced on `verify`
- Full Docker Compose: PostgreSQL 16, Kafka (Confluent 7.6), Zookeeper, App, Prometheus, Grafana

**How to apply:** When resuming this project, all source is in `/opt/development/git/personal/interviews/mindata-sling-backend-engineer-test/`.
