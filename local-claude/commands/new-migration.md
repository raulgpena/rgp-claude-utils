You are creating a new Liquibase database migration.
Migration description: $ARGUMENTS

Follow ALL rules in CLAUDE.md — especially the Liquibase section.

## Steps

### 1. Determine next changeset ID
- Scan `src/main/resources/db/changelog/changes/` for the highest existing number
- Increment by 1
- Format: `{YYYY}-{NNN}-$ARGUMENTS`
  Example: `2025-005-add-customer-table`

### 2. Determine current year folder
Create folder `src/main/resources/db/changelog/changes/{YYYY}/` if it doesn't exist.

### 3. Create the changeset file
File: `src/main/resources/db/changelog/changes/{YYYY}/{NNN}-$ARGUMENTS.yaml`

Template to follow:
```yaml
databaseChangeLog:
  - changeSet:
      id: {YYYY}-{NNN}-$ARGUMENTS
      author: dev-team
      comment: "Description of what this migration does and why"
      changes:
        - createTable:          # or addColumn, createIndex, etc.
            tableName: ...
            columns:
              - column:
                  name: id
                  type: UUID
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: created_at
                  type: TIMESTAMP WITH TIME ZONE
                  defaultValueComputed: NOW()
                  constraints:
                    nullable: false
              - column:
                  name: updated_at
                  type: TIMESTAMP WITH TIME ZONE
                  defaultValueComputed: NOW()
                  constraints:
                    nullable: false
      rollback:
        - dropTable:
            tableName: ...   # mirror the forward change exactly
```

### 4. Column type mapping
Use these PostgreSQL types:
| Java type       | Liquibase type                  |
|-----------------|---------------------------------|
| UUID            | UUID                            |
| String          | VARCHAR(N)                      |
| LocalDate       | DATE                            |
| Instant         | TIMESTAMP WITH TIME ZONE        |
| BigDecimal      | NUMERIC(19,4)                   |
| Boolean         | BOOLEAN                         |
| Enum            | VARCHAR(50)                     |
| Long/Integer    | BIGINT / INT                    |

### 5. Index conventions
If the migration adds a table or foreign key, always add indexes:
- PK index is automatic
- FK columns: `idx_{table}_{fk_column}`
- Frequently queried columns: `idx_{table}_{column}`
- Unique constraints: `uq_{table}_{column}`

### 6. Register in master changelog
Add an include entry to `db.changelog-master.yaml`:
```yaml
- include:
    file: changes/{YYYY}/{NNN}-$ARGUMENTS.yaml
    relativeToChangelogFile: true
```

### 7. Validation checklist
Before finishing, verify:
- [ ] Changeset ID is unique and follows naming convention
- [ ] Every `change` has a corresponding `rollback`
- [ ] All NOT NULL columns have a `defaultValue` or are new tables
- [ ] Index names follow `idx_{table}_{column}` convention
- [ ] Master changelog updated
- [ ] No modification of existing changesets

Print the final file content and the updated master changelog entry.

