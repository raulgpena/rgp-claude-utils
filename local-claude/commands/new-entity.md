You are adding a new domain entity to the current microservice.
Entity name: $ARGUMENTS

Infer the current service from the working directory.
Follow ALL rules in CLAUDE.md — especially the hexagonal layer rules.

## Steps — execute in order

### 1. Domain model
Create `domain/model/$ARGUMENTS.java`:
- Java record if immutable, class if it has behaviour
- UUID id field typed as `$ARGUMENTS.Id` (nested record value object)
- Include `createdAt` and `updatedAt` as `Instant` fields
- Add domain methods for business rules (validate, change state, etc.)
- No Spring annotations. No JPA annotations.

### 2. Domain exception
Create `domain/model/$ARGUMENTSNotFoundException.java` extending DomainException.
Create `domain/model/Invalid$ARGUMENTSException.java` extending DomainException.

### 3. Driving port (use cases)
Create interfaces in `domain/port/in/`:
- `Create$ARGUMENTSUseCase.java`  → `execute(Create$ARGUMENTSCommand): $ARGUMENTS`
- `Find$ARGUMENTSUseCase.java`    → `findById($ARGUMENTS.Id): $ARGUMENTS`
- `Update$ARGUMENTSUseCase.java`  → `execute(Update$ARGUMENTSCommand): $ARGUMENTS`
- `Delete$ARGUMENTSUseCase.java`  → `execute($ARGUMENTS.Id): void`

Create command records in `domain/port/in/`:
- `Create$ARGUMENTSCommand.java` (Java record, Bean Validation annotations OK)
- `Update$ARGUMENTSCommand.java` (Java record)

### 4. Driven port (repository)
Create `domain/port/out/$ARGUMENTSRepository.java`:
- `save($ARGUMENTS): $ARGUMENTS`
- `findById($ARGUMENTS.Id): Optional<$ARGUMENTS>`
- `findAll(Pageable): Page<$ARGUMENTS>`
- `deleteById($ARGUMENTS.Id): void`
- `existsById($ARGUMENTS.Id): boolean`

### 5. Application service
Create `application/usecase/$ARGUMENTSService.java`:
- Implements all 4 use-case interfaces
- `@Service` + `@Transactional`
- Constructor injection of $ARGUMENTSRepository port
- Record metrics via $ARGUMENTSMetrics (create this class too)

### 6. JPA adapter
Create in `adapter/out/persistence/entity/$ARGUMENTSJpaEntity.java`:
- Full JPA entity with `@Entity`, `@Table`
- UUID PK with `@GeneratedValue(strategy = UUID)`
- `@EntityListeners(AuditingEntityListener.class)`

Create `adapter/out/persistence/repository/$ARGUMENTSJpaRepository.java`:
- Extends `JpaRepository<$ARGUMENTSJpaEntity, UUID>`

Create `adapter/out/persistence/$ARGUMENTSPersistenceAdapter.java`:
- Implements `$ARGUMENTSRepository` port
- Uses MapStruct mapper (create `$ARGUMENTSMapper.java`)

### 7. Redis cache adapter
Create `adapter/out/cache/$ARGUMENTSCacheAdapter.java`:
- Key pattern: `{service-name}:$ARGUMENTS:{id}` (lowercase)
- TTL: 10 minutes default
- Methods: get, set, evict

### 8. REST adapter
Create in `adapter/in/web/`:
- `$ARGUMENTSController.java` — full CRUD, `/api/v1/{lowercase-plural}`
- `$ARGUMENTSRequest.java`   — Java record + Bean Validation
- `$ARGUMENTSResponse.java`  — Java record
- `$ARGUMENTSMapper.java`    — MapStruct interface (web layer)

### 9. Liquibase migration
Create `db/changelog/changes/{year}/NNN-create-{lowercase}-table.yaml`
with the full table DDL, primary key, timestamps, and rollback section.

### 10. Tests
Create:
- `domain/service/$ARGUMENTSServiceTest.java` — pure unit test, no Spring
- `adapter/out/persistence/$ARGUMENTSPersistenceAdapterTest.java` — @DataJpaTest + Testcontainers
- `adapter/in/web/$ARGUMENTSControllerTest.java` — @WebMvcTest

## When done
Print a summary table: file path → layer → purpose.

