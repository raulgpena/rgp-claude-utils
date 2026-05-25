# CLAUDE.java.md — Java Conventions

## Stack
- Java 21
- Spring Boot 3.x (Actuator, Spring Data JPA, Spring data Redis)
- Spring Cloud (Config, OpenFeign, k8s)
- Maven
- JUnit 5 + Mockito + AssertJ
- MapStruct (object mapping)
- OpenTelemetry

---

## Project Structure
```
src/main/java/com/company/<service>/
├── domain/
│   ├── model/          # Entities, Value Objects, Aggregates
│   ├── event/          # Domain Events
│   └── exception/      # Domain exceptions
├── application/
│   ├── usecase/        # Use case implementations
│   ├── port/
│   │   ├── in/         # Driving ports (interfaces for use cases)
│   │   └── out/        # Driven ports (Repository, Gateway interfaces)
│   └── dto/            # Request/Response DTOs
├── infrastructure/
│   ├── persistence/    # JPA entities, repositories (adapters)
│   ├── messaging/      # Kafka/RabbitMQ producers & consumers
│   └── client/         # Feign clients, RestTemplate adapters
└── interfaces/
    ├── rest/           # @RestController classes
    └── scheduler/      # @Scheduled tasks
```

---

## Java 21 — Use Modern Features
- Use **Records** for DTOs and Value Objects.
- Use **Sealed classes** for domain result types / ADTs.
- Use **Pattern matching** (`instanceof`, `switch`) where it improves clarity.
- Use **Virtual threads** (Project Loom) for I/O-bound workloads where applicable.
- Use **`var`** for local variables when type is obvious from the right side.
- Prefer **`Optional`** over returning `null`. Never return `Optional` from a field or constructor parameter.

## Spring Boot Rules
- Use **constructor injection** always — never field injection (`@Autowired` on fields is forbidden).
- Annotate use cases with `@Service`, adapters with `@Component` or `@Repository`/`@RestController`.
- Keep `@RestController` classes thin — they only validate input, call use case, and map response.
- Use `@ControllerAdvice` + `@ExceptionHandler` for centralized error handling.
- Externalize all config via `@ConfigurationProperties` (typed, validated with Bean Validation).
- Use **Spring profiles** (`dev`, `staging`, `prod`) — never `if` blocks checking environment names in code.
- Read the propterties files of each environment from k8s configmaps and secrets.
- All yml files must be documented
- All properties must be documented

## Persistence (JPA / Spring Data)
- JPA entities live in `infrastructure/persistence/` — they are **never** the same as domain entities.
- Use **MapStruct** to map between JPA entities and domain models.
- Avoid N+1 queries — use `@EntityGraph` or JPQL joins explicitly.
- Transactions belong in use cases (`@Transactional` on use case methods), not in repositories.
- Never use `ddl-auto: update` in all environments.

## Error Handling
- Define a hierarchy under a base `DomainException` in the domain layer.
- Use `@ControllerAdvice` to map domain exceptions → HTTP status codes.
- Log exceptions with SLF4J + structured arguments: `log.error("Order not found", kv("orderId", id))`.

## Testing
- Unit tests: JUnit 5 + Mockito + AssertJ. No Spring context — pure Java.
- Integration tests: `@SpringBootTest` + **Testcontainers** for real DB/broker.
- Use `@WebMvcTest` for controller-layer slice tests.
- Use `@DataJpaTest` for persistence slice tests.

## Code Style
- Follow **Google Java Style Guide**.
- Max line length: 120 characters.
- Use `final` on fields and parameters where possible to signal immutability intent.
- No `System.out.println` — always SLF4J logger.
- All classes must be documented with Javadoc. (Mehods and fields need it too):
  . The Javadoc tag @author with Raul Pena (raul.pena@gmail.com)
  . The Javadoc tag @since with jdk 21
  . The Javadoc tag @version with 1.0.0
  . The Javadoc classes header must be:
    ```
    /*
     * @className.java <date>
     * Copyright 2026 Momcorp, Inc. All rights reserved.
     * Momcorp/CONFIDENTIAL
     * */

    ```
  
## Project rules
- .gitignore for java applications.
