# Global Rules — Agora Ecosystem (Cursor)

## 1. Core Principles
- Aim for clarity over cleverness in all code.
- Follow these rules across all services.
- Prefer predictable patterns, modular architecture, and standardized naming.
- All agents must document assumptions, reasoning, and side effects.
- Never silently break existing functionality without explanation.


## 2. Language & Framework Standards
### Micro Services
- Use TypeScript (strict mode always enabled).
- Favor Express or TSOA depending on service style.
- All API Routes must be documented with OpenAPI Standards
- Services must follow a `controllers → services → repositories` pattern.
- Use DTOs for all inbound/outbound API traffic.
- All requests and responses must have a typed interface.

### Frontend
- Use **Next.js 14+** with **TypeScript**.
- Prefer the **App Router** and **Server Components** where appropriate; use Client Components only when needed (hooks, browser APIs, interactive state).
- Organize UI into:
  - `app/` for routing and layouts
  - `app/(routes)/<feature>` for feature-based routes
  - `components/` for shared, reusable components
  - `lib/` for frontend utilities, API clients, and helpers
- State management:
  - Prefer **React hooks + lifted state** for local/simple cases.
  - Use **Zustand** or **Context** for global/shared state where needed.
- API communication:
  - Use a **typed API client** (Axios) placed in `lib/api/`.
  - All API calls must use shared TypeScript types/interfaces for requests and responses.
- Styling:
  - Prefer **Tailwind CSS** for layout and styling.
  - Use component-level patterns (e.g. design system components in `components/ui`).
- Forms:
  - Use **react-hook-form** with Zod/Yup validation schemas.
  - Centralize common form components and validation helpers.
- Routing rules:
  - Use **nested layouts** to share UI structure (e.g. dashboard shell, public vs. authenticated layouts).
  - Keep routes REST-like and semantic (e.g. `/dashboard/stores`, `/dashboard/products`, `/settings/profile`).
- Auth:
  - Integrate with **Supabase Auth** using Next.js middleware and server-side session helpers.
  - Protect routes via:
    - Route Handlers with auth checks
    - Layout-level guards for authenticated sections.
- Performance:
  - Use **dynamic imports** for heavy components.
  - Cache with `fetch` + `next` configuration or React cache patterns where appropriate.
  - Optimize images using `next/image`.

### Testing

- **Test Runners**
  - Use **Vitest** code where possible.

- **Directory & File Structure**
  - Co-locate tests near the code they cover in a `__tests__` directory:
    - `src/<module>/__tests__/*.test.ts`
  - Test file naming:
    - Unit tests: `*.test.ts`
    - Integration tests: `*.int.test.ts`
    - E2E/API tests (if present in a service): `*.api.test.ts`

- **Types of Tests**
  - **Unit Tests**
    - Focus on a single function, class, or method.
    - No network calls, no real databases.
    - Use mocks/stubs for I/O, external services, and side effects.
  - **Integration Tests**
    - Test multiple layers together (e.g. controller + service + repository with a real or test DB).
    - Prefer running against a **test database** (Dockerized Postgres/Mongo/Neo4j).
  - **E2E/API Tests** (optional per service)
    - Hit HTTP endpoints via supertest or similar.
    - Use isolated test environments and reset state between suites.

- **Coverage Expectations**
  - Aim for:
    - **80%+ line coverage** per service.
    - Critical domains (auth, payments, identity, permissions) should trend higher.
  - Coverage is a guideline; prioritize meaningful, behavior-focused tests.

- **Mocking & Test Doubles**
  - Use the built-in mocking tools from Vitest.
  - Only mock:
    - Network calls
    - External APIs (e.g. Auth0, payment processors, 3rd party services)
    - Message bus (RabbitMQ) publishers/subscribers in unit tests
  - Do **not** mock your own business logic if it is the subject under test.

- **Database & State Management in Tests**
  - Use separate **test databases** for Postgres, Mongo, and Neo4j.
  - Before each test suite:
    - Run migrations or apply test schema.
  - Before/after each test (or suite):
    - Clear or reset collections/tables.
  - Prefer **Docker Compose**-based test environments checked into the repo (e.g. `docker-compose.test.yml`).

- **Testing Conventions**
  - Tests must:
    - Be deterministic and not rely on timezones, external network, or randomness (unless seeded).
    - Describe behavior clearly using `describe`/`it` blocks:
      - Example: `it("creates a store with default settings when none provided", ...)`.
    - Assert on behavior and state, not implementation details.
  - When a bug is found, first:
    - Write a failing test that reproduces the bug.
    - Then fix the bug and ensure the test passes.

- **API Testing Rules**
  - Use supertest or similar tools for HTTP endpoint tests.
  - Always verify:
    - Status codes
    - Response shape (types, required fields)
    - Error cases (validation errors, unauthorized, forbidden, not found).
  - For contract-critical APIs, consider snapshot tests for response structure (not for volatile values).

- **CI Integration**
  - All services must have a `test` script in `package.json`:
    - `"test": "vitest --runInBand"` or `"test": "jest"`
  - CI must:
    - Run tests on every PR.
    - Fail the build on test failures or coverage dropping below thresholds (if configured).
  - Long-running or heavy integration suites can be tagged and run on a separate pipeline (e.g. nightly).

- **Documentation for Tests**
  - Each service should include in its README:
    - How to run unit tests.
    - How to run integration/E2E tests.
    - Any required environment variables or Docker commands for test setup.

---

## 3. Microservice Architecture Rules
- Each service must be independently deployable.
- Never assume another service’s database schema. Communicate only via:
  - REST (internal network-only)
  - RabbitMQ events
  - JWT claims (from the authenticated user context)
- All inter-service communication must be typed, versioned, and documented.
- Services must treat the JWT as the source of truth for:
  - `sub` (user id)
  - roles / permissions
  - tenant / store membership
  - other authorization-relevant claims
---

## 4. Database Rules
- Identity/Auth → Supabase/Postgres  
- Product / Store / Inventory / Commerce services → MongoDB  
- AI, relationships, memory, recommendations → Neo4j  

---

## 5. Event Bus Rules (RabbitMQ)
### 5.1 Exchange & Routing Conventions

All services must publish domain events through **topic exchanges** using the naming pattern:

`agora.<service>.<domain>`

Routing keys must follow this structure:

`<service>.<domain>.<action>`

**Examples**
- `auth.user.created`
- `store.inventory.updated`
- `product.item.published`
- `checkout.order.completed`

Each domain service must define:
- one topic exchange per domain  
- queues bound with routing patterns (e.g. `store.inventory.*`)

---

### 5.2 Message Structure

Every event message must include the following fields:

```json
{
  "eventId": "uuid",
  "timestamp": "ISO-8601",
  "version": 1,
  "producer": "product-service",
  "routingKey": "product.item.published",
  "payload": {
    // typed event data
  }
}
```

**Rules**
- `eventId` must be globally unique (UUID v4).
- `version` increments when the event schema changes.
- `producer` identifies the service emitting the event.
- `payload` must be fully typed via TypeScript interfaces.

---

### 5.3 Queue Rules

- Each service owns and consumes from **its own queues**.
- No shared queues between services.
- Queue names must follow this pattern:

`<service>.<domain>.<purpose>-queue`

**Examples**
- `inventory.stock.updated-queue`  
- `billing.payment.process-queue`

---

### 5.4 Consumer Rules

Consumers must:

- Acknowledge messages **only after successful processing**.
- Retry transient failures.
- Reject and dead-letter messages that fail after max retries.

Each service must define a **DLX (Dead Letter Exchange)** using this pattern:

`<service>.<domain>.dlx`

Dead-letter messages must contain:
- the original message  
- error details  
- retry count  

---

### 5.5 Retry Rules

Retries must be implemented via queue TTL + DLX routing.

**Default recommendation**
- **3 retry attempts**
- Exponential backoff between retries

After maximum retries, messages are routed to a **DLQ**:

`<service>.<domain>.dlq`

Services must alert on increasing DLQ size.

---

### 5.6 Auth & Identity in Events

Optional metadata may be included in headers:

- `x-user-id: <uuid>`
- `x-store-id: <uuid>`
- `x-service-name: <producer-service>`
- `x-correlation-id: <uuid>`

**Rules**
- Consumers must verify that the producing service is authorized to emit events with that routing key.
- Do **not** include raw JWTs or sensitive identity data in message bodies.

---

### 5.7 Versioning & Schema Governance

- Each event type must have a TypeScript interface stored in `/events`.
- Breaking changes require incrementing `version`.
- Services should publish the latest version by default.
- If backward compatibility is required, consumers may handle multiple versions explicitly.

---

### 5.8 Observability & Logging

Every publish and consume operation must be logged with:

- `routingKey`  
- `eventId`  
- producer / consumer  
- processing time  
- result (`success` / `failure` / `retry` / `dead-letter`)

On failure:
- log the error reason  
- log `correlationId`  
- do **not** log sensitive payload data  

---

### 5.9 Performance & Connection Rules

- Use a **single shared connection** per service, not one per publish.
- Use separate **channels** for publishers and consumers.
- Batch publishes where appropriate.
- Never use auto-ack; always explicitly `ack` or `nack`.

---

### 5.10 Example Summary

**Exchange**  
`agora.product.events`

**Routing key**  
`product.item.published`

**Queue**  
`inventory.product.published-queue`

**Message**

```json
{
  "eventId": "4f0f09ff-b077-4c46-8003-bdd1f399b7d1",
  "timestamp": "2025-01-01T12:00:00Z",
  "version": 2,
  "producer": "product-service",
  "routingKey": "product.item.published",
  "payload": {
    "productId": "123",
    "name": "Example Product",
    "status": "published"
  }
}
```
---

## 6. Naming Conventions

### Files & Folders
Standard service layout:

/src
/controllers
/docs
/services
/repositories
/models
/routes
/events
/utils


### Databases
- supabase: snake_case  
- Mongo/Mongoose: camelCase  
- Neo4j: PascalCase nodes, UPPERCASE_RELATIONSHIPS  

### APIs
REST endpoint format:

/api/{version}/<service>/<resource>


---

## 7. Documentation Rules
- Every service must have a `/docs` folder.
- All APIs must include OpenAPI/Swagger specs.
- Any new domain concept must be added to `/docs/AGORA_MASTER_PLAN.md`.

---

## 8. Code Quality & Review Rules
- Any generated file must:
  - Include an explanation header.
  - Follow linting rules.
  - Use types/interfaces for all data structures.
- Prefer dependency injection where possible.
- Avoid global state and magic values.
- Avoid mutating shared objects.

---

## 9. AI Agent Behavior Rules
- All agents must follow the BMAD method (Behavior → Model → Actions → Data).
- Agents stay within their assigned roles unless explicitly instructed otherwise.
- Reference global rules before generating or updating code.
- Output must be deterministic and reproducible.
- Use structured formatting (Markdown, code blocks, diffs, etc.).

---

## 10. File Generation Rules

### When generating files:
- Create the full path if it doesn't exist.
- Include imports and exports.
- Include clear instructions on where the file fits in the architecture.
- Output code that runs immediately without manual fixing.

### When modifying files:
- Use diff-style output.
- Never rewrite entire files unless explicitly allowed.

---

## 11. Error Handling Rules
- Use typed errors (`AppError`, `ValidationError`, etc.).
- Never expose internal error messages in public API responses.
- Log all errors in a consistent JSON format.

---

## 12. Security Rules
- Authentication handled via Supabase JWTs with scope checks.
- All services must validate inputs using Zod.
- Never expose database connections or internals over public endpoints.
- Only HTTPS for any external-facing API.

---

## 13. Service-to-Service Authentication

### 13.1 Core Principles
- Treat **every service call as untrusted by default**, even on the internal network.
- Distinguish clearly between:
  - **User → Service** auth (end-user JWT, from Auth0/Supabase/etc.)
  - **Service → Service** auth (internal service identity)
- No service should trust another service **just because** it’s on the same VPC, subnet, or cluster.

---

### 13.2 Token Types

We use two main token types:

1. **User JWT**
   - Issued by the external IdP (Auth0/Supabase/etc.).
   - Represents a human user.
   - Used for authorization decisions (roles, permissions, store memberships, etc.).
   - Contains claims such as:
     - `sub` (user id)
     - `email`
     - `roles` / `permissions`
     - `tenant` / `storeId` (where applicable)

2. **Service JWT (Internal Service Token)**
   - Issued by an internal signing key (service-auth).
   - Represents a **service identity**, not a user.
   - Used when one service talks directly to another.
   - Contains claims such as:
     - `sub`: service name (e.g. `order-service`)
     - `iss`: `service-auth`
     - `aud`: target service name or logical audience (e.g. `inventory-service`)
     - `scopes`: allowed operations (e.g. `inventory.read`, `inventory.update`)
     - `exp`: short-lived expiry (e.g. 5–15 minutes)

---

### 13.3 HTTP Service-to-Service Calls

- All internal HTTP calls between services **must** include an `Authorization` header:
  - `Authorization: Bearer <service-jwt>`
- The **called service** must:
  - Verify the token signature using the internal `service-auth` key.
  - Check:
    - `iss` is `service-auth`
    - `aud` matches its own service name or a trusted audience
    - `exp` is valid (not expired)
    - `scopes` include the required permission for the action
  - Reject requests with invalid/missing tokens with `401` or `403`.


#### 13.3.1 Propagating User Context

When a request originates from a real user:

- The **edge/API gateway** validates the **User JWT**.
- Downstream services may receive:
  - The **User JWT** (as `x-user-jwt`), or
  - A reduced, signed **User Context** object (user id, roles, tenant) to avoid token bloat.
- Services must:
  - Use the **Service JWT** to authenticate the caller service.
  - Use the **User JWT/User Context** for authorization decisions (e.g. “is this user allowed to modify this store?”).

---

### 13.4 Message Bus (RabbitMQ) Authentication

- Each service has its **own RabbitMQ user** and/or **vhost**:
  - Example users: `orders_svc`, `inventory_svc`, `billing_svc`.
- Permissions are granted per service:
  - Which exchanges/queues it can `read` from.
  - Which exchanges/queues it can `write` to.
- Services must not share RabbitMQ credentials.

For messages:

- Identify the producer service via:
  - Connection identity (RabbitMQ user), and/or
  - Message headers such as:
    - `x-service-name: order-service`
    - `x-correlation-id`
- Sensitive operations should:
  - Include user context in headers (user id, tenant, roles) when relevant.
  - Be validated by the consumer (e.g. check that the producing service is allowed to trigger that event).

---

### 13.5 Network-Level Security

- All service-to-service traffic (HTTP + RabbitMQ) must be restricted to **private networks**.
- Only API gateways / edge services are internet-facing.
- If/when available (e.g. service mesh, cloud LB):
  - Prefer **mTLS** between services to provide:
    - Transport encryption
    - Mutual service identity at the connection level

---

### 13.6 Key Management

- Service JWTs are signed using an internal `service-auth` keypair (or shared secret if symmetric).
- Signing keys must be stored in:
  - A secrets manager (Vault, cloud secrets, etc.), not in the code repo.
- Key rotation:
  - Support multiple keys via `kid` (key id) in JWT header.
  - Services must be able to verify tokens signed with the current and previous active keys.

---

### 13.7 Implementation Rules

- No service call should be accepted **without**:
  - A valid Service JWT (or mTLS at the mesh level), and
  - Appropriate scope/permissions for the requested operation.
- Do not rely on:
  - IP whitelisting
  - Hostname alone
- Log all failed auth attempts with:
  - service name
  - endpoint/queue
  - reason (invalid signature, expired token, missing scope, etc.)

---

## 14. Observability Rules (OpenTelemetry)

### 14.1 Core Principles
- All services must emit **traces, metrics, and logs** using OpenTelemetry SDKs.
- Use consistent **semantic conventions** across the entire ecosystem.
- Observability is mandatory, not optional — instrument while building, not after.
- Traces must allow a developer to follow a request across all services end-to-end.

---

### 14.2 Tracing Requirements
Every service must:
- Create a **root span** for all inbound HTTP requests.
- Propagate OTEL context (`traceparent`, `tracestate`) through:
  - internal REST calls
  - RabbitMQ publish/consume
  - async tasks
- Wrap critical operations in spans:
  - DB queries
  - External API calls
  - Event publishing
  - Event consumption
  - Cache operations

**Span attributes must include:**
- `service.name`
- `service.version`
- `http.method`, `http.route`, `http.status_code` (for HTTP spans)
- `messaging.system="rabbitmq"` for messaging spans
- `db.system`, `db.statement`, `db.duration_ms`
- `error.type`, `error.message`, `error.stack` when applicable

---

### 14.3 Metrics Requirements
All services must emit base metrics using OTEL instruments.

**Required metrics:**
- `http.server.request.count`
- `http.server.request.duration`
- `messaging.consumer.processed`
- `messaging.consumer.failed`
- `messaging.consumer.retry`
- `messaging.consumer.deadletter`
- `db.query.count`
- `db.query.duration`

---

### 14.4 Logging Requirements
Use **structured JSON logging** with OTEL correlation.

Every log entry must include:
- `trace_id`
- `span_id`
- `service.name`
- `timestamp`
- `level`
- `message`

**Rules:**
- Never log secrets, passwords, or raw tokens.
- Use levels appropriately:
  - `FATAL` = Represents a severe error that likely leads to application termination or a complete system failure, requiring immediate attention.
  - `ERROR` = Signifies a problem or failure in a specific function or component that prevents it from performing its intended task.
  - `WARN`  = Indicates a potentially harmful situation or an unexpected event that doesn't immediately cause a failure but might lead to problems.
  - `INFO`  = Logs general information about the normal operation and significant events within the application.
  - `DEBUG` = Offers detailed diagnostic information useful during development and debugging.
  - `TRACE` = Provides the most fine-grained details, often capturing step-by-step execution and internal states of the application.

---

### 14.5 Instrumentation Standards
All services must use shared wrappers (utility libraries) for:

- HTTP clients (Axios with OTEL interceptors)
- Database clients (Prisma, Mongoose, Neo4j driver)
- RabbitMQ producers/consumers

Manual spans must be added around important business logic.

Auto-instrumentation should be used where possible.

---

### 14.6 RabbitMQ Instrumentation

**On Publish:**
- Create span: `messaging.publish`
- Required attributes:
  - `messaging.destination` (exchange)
  - `messaging.rabbitmq.routing_key`
  - `messaging.operation="publish"`

**On Consume:**
- Create span: `messaging.consume`
- Required attributes:
  - `messaging.consumer.queue`
  - `messaging.rabbitmq.routing_key`
  - `messaging.operation="process"`

**Context propagation:**
- Trace context must be added to message headers.

---

### 14.7 Exporter Requirements

All telemetry must be sent to a shared **OpenTelemetry Collector** using OTLP.

Collector responsibilities:
- batching
- filtering
- sampling
- sending data to:
  - Grafana / Tempo
  - Coralogix
  - Datadog
  - or any chosen backend

Collectors must run both locally (dev) and in production clusters.

---

### 14.8 Sampling Rules

**Production:**
- Parent-based sampling
- 5–10% sampling rate
- Always sample error spans

**Staging:**
- 25–50%

**Development:**
- 100%

Dynamic sampling (optional):
- raise sampling during incidents
- raise sampling for specific services/routes

---

### 14.9 Dashboards & Alerts

**Required dashboards:**
- HTTP latency (p50, p95, p99)
- Error rate by route
- DB query duration distribution
- RabbitMQ event throughput
- DLQ growth over time
- Event processing success/failure

**Required alerts:**
- Latency spikes
- Error rate spikes
- High DB latency
- Queue backlog growth
- DLQ threshold breach

---

### 14.10 Correlation Requirements
All services must ensure:

- Every log includes `trace_id` + `span_id`.
- Every HTTP request forwards OTEL trace headers.
- Every RabbitMQ message includes OTEL context in headers.
- Every event handler continues the trace if context exists.

This allows complete end-to-end visibility across distributed systems.

# End of Global Rules