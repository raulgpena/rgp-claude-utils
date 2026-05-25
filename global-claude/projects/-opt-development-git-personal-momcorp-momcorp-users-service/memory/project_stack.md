---
name: Project Stack and Structure
description: Core stack, package layout, and architectural decisions for momcorp-users-service
type: project
originSessionId: 297d1590-00d2-4d87-90bf-20ce35d28168
---
Java 21 + Spring Boot 3.3.x + Spring Cloud 2023.0.x (Config Client) + Spring Data JPA + PostgreSQL + Spring Security (JWT) + Spring Actuator + OpenTelemetry (Micrometer bridge).

Maven build. Package root: `com.momcorp.usersservice`.

**Why:** Scaffolded from scratch May 2026. User specified the full stack; no Eureka/Feign — Config Client only.

**How to apply:** When suggesting dependencies or patterns, stay within this stack. OTel is via micrometer-tracing-bridge-otel + opentelemetry-exporter-otlp, not the OTel Java agent (noted as an alternative in CLAUDE.md). JWT uses JJWT 0.12.x API (Jwts.builder(), Jwts.parser().verifyWith()). Local dev uses `application-local.yml` with ddl-auto=create-drop and no Config Server.
