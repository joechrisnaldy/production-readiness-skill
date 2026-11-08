---
name: production-readiness
description: Use for code reviews, security or reliability audits, and production readiness checks, and before designing or changing authentication, data models, payments, external integrations, background jobs, caching, or deployment. Contains a 150-item checklist covering security, data, concurrency, resilience, messaging, networking, caching, performance, APIs, observability, testing, delivery, disaster recovery, client side, and LLM features.
---

# Production Readiness Checklist

Each entry states what to check. Apply what fits the project's stage using the stage tiers in CLAUDE.md; if none are defined, infer the stage (Prototype, MVP, Production, or Scale) and state it. In reviews, mark entries that do not fit the stage as not applicable yet, with the trigger that would make them relevant.

## 1. Security and access control

- **Authorization:** Check permissions server-side on every endpoint and for every object accessed, not only at login (prevents IDOR and BOLA). With Supabase, Firebase, or similar backends, row-level security or security rules must be enabled and tested.
- **Input Validation:** Validate type, length, format, and range at every trust boundary with a schema library (Zod, Pydantic, or equivalent). Client-side validation is for UX, never for security.
- **SQL Injection:** Use parameterized queries or ORM bindings only. Allowlist dynamic identifiers such as sort columns.
- **XSS:** Rely on framework escaping. Avoid `innerHTML`, `dangerouslySetInnerHTML`, and `v-html` with untrusted data, or sanitize with a vetted library such as DOMPurify. Add a Content Security Policy.
- **CSRF:** For cookie-based sessions, use SameSite cookies plus CSRF tokens or Origin checks on state-changing requests. Never change state on GET.
- **CORS:** Allowlist exact origins. Never reflect arbitrary origins or allow a wildcard with credentials. CORS is not access control.
- **SSRF:** When the server fetches user-supplied URLs, allowlist destinations, block private, loopback, and link-local addresses for IPv4 and IPv6 (including cloud metadata at `169.254.169.254`), and recheck after redirects and DNS resolution.
- **Secrets Management:** Load secrets from environment variables or a secrets manager. They never appear in code, logs, container images, or git history, and anything that leaked gets rotated.
- **IAM:** Apply least privilege to every person, service, and token. Use separate credentials per environment and per service, and never use root or admin keys in applications.
- **OAuth:** Use a proven provider or library rather than custom auth. Use Authorization Code with PKCE for browser and mobile clients, exact redirect URI matching, state validation, and minimal scopes.
- **JWT Rotation:** Use short-lived access tokens, rotating refresh tokens with reuse detection, and signing keys rotated through key IDs (`kid`) and JWKS. Always verify signature, algorithm, issuer, audience, and expiry.
- **Password and Session Security:** Hash passwords with Argon2id or bcrypt. Set session cookies as HttpOnly, Secure, and SameSite. Offer MFA where accounts hold value.
- **TLS:** Serve HTTPS everywhere with automatic certificate renewal, TLS 1.2 minimum (prefer 1.3), and HSTS. Never disable certificate verification.
- **Encryption in Transit:** Encrypt internal traffic as well as public traffic, including service, database, cache, and queue connections.
- **Encryption at Rest:** Encrypt databases, disks, backups, and object storage, with keys in a KMS. Consider field-level encryption for highly sensitive data.
- **Security Headers:** Set Content-Security-Policy (including frame-ancestors against clickjacking), Strict-Transport-Security, `X-Content-Type-Options: nosniff`, and Referrer-Policy.
- **WAF:** Place a web application firewall with managed rule sets in front of public production apps.
- **DDoS Protection:** Rely on provider protection (Cloudflare, AWS Shield, or platform defaults) plus rate limits and autoscaling caps, so an attack cannot become an unlimited bill.
- **Supply Chain Security:** Run dependency vulnerability scans (npm audit, pip-audit, Dependabot, or similar) in CI. Verify every new package is real, correctly named, and maintained.
- **File Uploads:** Check type by content rather than extension, cap size, store in object storage under random names, and serve private files through short-lived signed URLs.
- **Audit Logs:** Record who did what and when for sensitive actions (auth events, permission changes, data exports, money movement) in append-only storage.
- **Privacy and PII:** Collect the minimum, know where personal data lives, keep it out of logs, define retention, and support deletion and export where laws such as GDPR or CCPA apply.

## 2. Data and databases

- **Database Indexing:** Index columns used in filters, joins, and sorts on tables that will grow, including foreign keys. Avoid redundant indexes that slow writes.
- **Query Optimization:** Check plans with `EXPLAIN` for slow or frequent queries. Select only needed columns, and avoid wrapping indexed columns in functions inside WHERE clauses.
- **N+1 Queries:** Find loops that query once per item. Replace them with joins, eager loading, or batching.
- **Connection Pooling:** Size pools to database limits. In serverless, use a pooler (PgBouncer, RDS Proxy, or the provider's pooler) because every instance opens its own connections.
- **Transactions:** Wrap multi-step writes that must succeed or fail together in one transaction, and choose the isolation level deliberately.
- **Constraints:** Enforce integrity in the database with NOT NULL, UNIQUE, foreign keys, and CHECK constraints. Application checks alone lose to race conditions.
- **Pagination:** Paginate every list endpoint and query with a capped page size. Prefer cursor (keyset) pagination for large or changing data.
- **Money and Time:** Store money as integer minor units or a decimal type. Store timestamps in UTC and convert only for display.
- **Read Replicas:** Add them only when reads overload the primary, and handle replication lag, especially when users read their own writes.
- **Replication:** Know whether it is synchronous or asynchronous and how much data a failover can lose.
- **Partitioning:** Partition very large tables (often by time) to keep indexes and maintenance manageable, and expire old data by dropping partitions.
- **Sharding:** Treat it as a last resort after indexing, caching, replicas, and vertical scaling. Pick a shard key that avoids hot spots and cross-shard queries.
- **Database Migrations:** Keep migrations versioned, reviewed, automated in deploys, and reversible where possible. For zero downtime, use expand and contract: add, backfill, switch, then remove in a later release. Back up before destructive changes.
- **Schema Versioning:** Version data contracts (database schemas, event payloads, API schemas) so old and new code can run side by side during deploys.

## 3. Concurrency and distributed systems

- **Race Conditions:** Hunt for check-then-act code (balances, inventory, unique usernames, coupon redemption). Fix it with atomic updates, constraints, transactions, or locks.
- **Thread Safety:** Synchronize or eliminate shared mutable state across threads, workers, and async tasks. Module-level globals are often shared across requests.
- **Optimistic Locking:** Check a version column or ETag on update for low-contention records, and return 409 or 412 on conflict.
- **Pessimistic Locking:** Use `SELECT ... FOR UPDATE` for high-contention critical sections, and keep locks short.
- **Deadlocks:** Acquire locks in a consistent order, keep transactions short, and retry on deadlock errors.
- **Distributed Locks:** Use them only for coordination across processes, with leases that expire and fencing tokens. Prefer database constraints or queues when those suffice.
- **Distributed Transactions:** Avoid two-phase commit across services. Keep transactional boundaries inside one database where possible.
- **Saga Pattern:** For workflows that span services, chain local transactions with compensating actions, and make every step idempotent.
- **CAP Theorem:** During a network partition, a system must choose consistency or availability. Decide per feature: balances and inventory need consistency, while feeds and counters can tolerate staleness.
- **Eventual Consistency:** Where it is accepted, design UI and logic for stale reads and make updates converge.
- **Leader Election:** When exactly one instance must act (schedulers, singleton workers), use a lease or a platform primitive, never a hardcoded instance.
- **Network Partitions:** Assume any call can fail midway or succeed without a response. Design with timeouts, idempotency, and reconciliation.
- **Clock Skew:** Never rely on clocks across machines for ordering or correctness. Use database timestamps, sequence numbers, or logical clocks, and allow small leeway in token expiry checks.
- **Service Discovery:** Resolve services through DNS, the platform, or a registry, never hardcoded IPs.

## 4. Resilience and fault tolerance

- **Timeouts:** Give every outbound call (HTTP, database, cache, queue, LLM API) connect and request timeouts that fit within the caller's own deadline.
- **Retries:** Retry only transient failures (timeouts, connection errors, 429, 502, 503, 504) on idempotent operations, with capped attempts. Avoid retrying at every layer at once, which multiplies load.
- **Exponential Backoff:** Back off exponentially with jitter, and honor `Retry-After` headers.
- **Circuit Breakers:** Stop calling a failing dependency after repeated errors, fail fast with a fallback, and probe for recovery.
- **Idempotency:** Give operations that may be retried (payments, orders, webhook handlers, jobs) an idempotency key or natural key, so duplicates have no extra effect.
- **Backpressure:** Bound queues, buffers, and concurrency. When overloaded, shed load or return 429 or 503 instead of accepting unbounded work.
- **Bulkheads:** Isolate resources (separate pools or workers) so one slow dependency cannot exhaust everything.
- **Graceful Degradation:** Decide what still works when a dependency is down, such as cached data, a read-only mode, or temporarily hiding a feature.
- **Graceful Shutdown:** On `SIGTERM`, stop accepting new work, finish in-flight requests and jobs, then close connections.
- **Health Checks:** Expose a cheap health endpoint for the load balancer or platform.
- **Liveness & Readiness Probes:** Liveness only detects a stuck process and must not depend on external services, or one database blip restarts everything. Readiness decides whether an instance should receive traffic and may check dependencies.
- **Failover:** Know what happens when the primary database, zone, or provider fails. Automate it where uptime matters, and test it.

## 5. Messaging, background work, and real time

- **Message Queues:** Move slow, retryable, or bursty work (emails, media processing, third-party calls) out of the request path into a queue with workers.
- **Delivery Semantics:** Assume at-least-once delivery, so consumers must be idempotent. Do not assume ordering unless the system guarantees it per key.
- **Dead Letter Queues:** Route messages that exhaust retries to a DLQ with an alert and a replay path. Never drop them silently or retry forever.
- **Pub/Sub:** Use it for fan-out to multiple independent consumers, with defined event schemas and ownership.
- **Event-Driven Architecture:** Adopt it when decoupling is worth the debugging cost. Treat events as versioned contracts and keep requests traceable across them.
- **Transactional Outbox:** When a database write must also publish an event, write both in one transaction (an outbox table) and relay asynchronously, which avoids the dual-write problem.
- **Cron Jobs:** Make scheduled jobs idempotent, safe to run twice or in overlap, limited to one instance at a time, and monitored for missed or failed runs.
- **WebSockets:** Authenticate on connect, send heartbeats, reconnect with backoff, and plan for multiple instances (shared pub/sub or sticky sessions).
- **Server-Sent Events:** Prefer them over WebSockets for one-way server to client streams such as notifications, progress, and LLM token streaming.
- **Long Polling:** Use it as a fallback when WebSockets and SSE are unavailable, with timeouts and a limit on held connections.
- **Webhooks:** For incoming webhooks, verify signatures (HMAC), reject stale timestamps to stop replays, acknowledge fast and process asynchronously, and dedupe by event ID. For outgoing webhooks, sign payloads, retry with backoff, and version event formats.

## 6. Traffic, edge, and networking

- **Rate Limiting:** Limit per user, API key, and IP on public, auth, and expensive endpoints (login, signup, password reset, OTP, LLM calls). Return 429 with `Retry-After`.
- **Load Balancing:** Spread traffic across health-checked instances. Keep app servers stateless so any instance can serve any request.
- **Reverse Proxies:** Terminate TLS, compress, and enforce body size and timeout limits at the proxy. Trust `X-Forwarded-For` only from known proxies so client IPs cannot be spoofed.
- **API Gateways:** Centralize auth, rate limiting, routing, and validation when there are multiple services or a public API. A single app rarely needs one.
- **CDN:** Serve static assets and media through a CDN with long cache lifetimes and content-hashed filenames.
- **DNS:** Lower TTLs before migrations, manage records as code where possible, and expect propagation delays.
- **TCP vs UDP:** Default to TCP-based protocols. Use UDP only for latency-sensitive traffic that tolerates loss (real-time media, games), with application-level handling.
- **HTTP/2 & HTTP/3:** Enable them at the edge or CDN for multiplexing and faster connections. This is usually configuration, not application code.
- **gRPC:** Use it for internal service calls that need strict contracts or streaming, with a deadline on every call. Browsers need gRPC-Web or a gateway.

## 7. Caching

- **Caching:** Cache only after measuring a need, and define the key, TTL, and audience of each cache. Never store user-specific or permission-dependent data in a shared cache under a generic key.
- **Edge Caching:** Set `Cache-Control` explicitly, mark personalized responses private or no-store, and use stale-while-revalidate where freshness allows.
- **Cache Invalidation:** Choose a strategy per cache (TTL, purge on write, or versioned keys). If stale data is unacceptable, invalidate on every change.
- **Cache Stampede:** Prevent thundering herds on expiry with request coalescing, locks, or jittered TTLs.

## 8. Performance, scaling, and cost

- **Latency:** Measure before optimizing, set budgets for key user paths, and run independent calls in parallel instead of in sequence.
- **Throughput:** Know the expected requests per second and the first bottleneck (database, CPU, third-party rate limits).
- **P99 Latency:** Track percentiles, not averages. One in 100 requests is slower than p99, and active users hit it often.
- **Tail Latency:** Fan-out amplifies slow outliers. Reduce fan-out, set timeouts, and consider hedged requests for critical reads.
- **Vertical Scaling:** Often the simplest first move. Know its ceiling and cost curve.
- **Horizontal Scaling:** Requires stateless app servers, with sessions in a shared store, uploads in object storage, and no in-memory state that must survive between requests.
- **Autoscaling:** Scale on meaningful signals (CPU, queue depth, latency) with minimum and maximum limits. The maximum protects your bill.
- **Memory Leaks:** Watch for unbounded caches, maps, listeners, and closures. Monitor memory over time and set container memory limits.
- **Garbage Collection:** Large heaps and heavy allocation churn cause pauses that surface as tail latency. Tune only after profiling.
- **Cold Starts:** In serverless, keep bundles small, lazy-load heavy dependencies, create clients outside the handler, and use provisioned concurrency for latency-critical paths.
- **Serverless Limits:** Know the platform's limits on execution time, payload size, memory, concurrency, and connections. Move long work to queues or workers.
- **Cost Optimization:** Right-size resources, add lifecycle rules for storage and logs, avoid paying for idle capacity, and review the largest line items.
- **Spend Controls:** Set billing alerts and budgets on cloud and third-party APIs (especially LLM APIs), plus per-user quotas, so a bug or an attack cannot create an unlimited bill.

## 9. APIs and versioning

- **API Versioning:** Never break existing clients. Version public APIs (path or header), add fields rather than changing their meaning, and deprecate with notice.
- **Semantic Versioning:** For libraries and packages, bump MAJOR for breaking changes, MINOR for features, and PATCH for fixes.
- **API Contracts:** Define request and response schemas (OpenAPI, GraphQL schema, or shared types) and validate against them.
- **Error Responses:** Use one consistent error format (RFC 9457 problem details is a good default) with correct status codes. Never leak stack traces, SQL, or internals to clients.
- **Request Limits:** Cap body size, upload size, array lengths, and query complexity, especially GraphQL depth and cost.

## 10. Observability

- **Logging:** Write structured (JSON) logs with level, timestamp, and a request or correlation ID. Log errors with context, and never log secrets or sensitive personal data.
- **Metrics:** Track RED for services (rate, errors, duration), USE for resources (utilization, saturation, errors), and key business events.
- **Monitoring:** Build dashboards for the golden signals (latency, traffic, errors, saturation) and add external uptime checks.
- **Error Tracking:** Capture unhandled exceptions with stack traces and release versions (Sentry or similar).
- **Alerts:** Alert on user-facing symptoms (error rate, latency, SLO burn) rather than every cause. Every alert must be actionable, routed to an owner, and linked to a runbook.
- **Distributed Tracing:** Propagate trace context across services and queues with OpenTelemetry once there is more than one service.
- **SLIs:** Define measurable indicators of user experience, such as availability, latency at a percentile, and freshness.
- **SLOs:** Set targets on SLIs that match user needs, for example 99.9% of requests succeed over 30 days.
- **Error Budgets:** Treat the unreliability an SLO allows as a budget. When it is spent, reliability work takes priority over features.
- **Observability:** The goal of everything above is answering "why is this broken?" from telemetry alone, without shipping new code to debug.

## 11. Testing and code quality

- **Automated Tests:** Write unit tests for logic, integration tests against a real database for data access, and end-to-end tests for critical flows such as signup, login, and checkout.
- **Regression Tests:** Ship every bug fix with a test that would have caught the bug.
- **Edge Cases:** Test empty, null, oversized, Unicode, duplicate, concurrent, and malicious inputs, not just the happy path.
- **Load Testing:** Before launches or expected spikes, test realistic traffic (k6, Locust, or similar) to find the first bottleneck.
- **Static Analysis:** Run linting, formatting, type checking, and SAST in CI. Fix warnings instead of suppressing them.

## 12. Delivery, deployment, and infrastructure

- **CI/CD:** Run build, lint, type checks, tests, and security scans on every change before merge. Keep deploys automated and repeatable, never from a laptop.
- **Environments and Configuration:** Separate development, staging, and production with their own credentials and data. Read configuration from the environment, and keep staging close to production.
- **Docker:** Use pinned base images, multi-stage builds, small images, a non-root user, and a `.dockerignore`. Never bake secrets into images or layers.
- **Kubernetes:** Adopt it only when scale and team size justify the operational cost; managed platforms (Cloud Run, ECS, Fly.io, Render, Vercel, or similar) suit most apps. If used, set resource requests and limits, probes, and disruption budgets.
- **Helm Charts:** On Kubernetes, template manifests with Helm (or Kustomize), version the charts, and keep per-environment values separate.
- **Infrastructure as Code:** Define cloud resources in code, reviewed and versioned like application code. No manual console changes in production.
- **Terraform:** If you use it (or OpenTofu, Pulumi, or CloudFormation), keep remote state with locking, review plans in CI before applying, pin provider versions, and never commit state files, since they can contain secrets.
- **Feature Flags:** Separate deploy from release. Ship dark, roll out gradually, and remove stale flags.
- **Rolling Deployments:** Replace instances gradually behind readiness checks. Old and new versions run side by side, so changes must be backward compatible.
- **Blue-Green Deployment:** Run two environments with an instant traffic switch and an instant switch back. Database changes must work with both versions.
- **Canary Releases:** Send a small share of traffic to the new version, compare errors and latency, then promote or abort automatically.
- **Rollbacks:** Give every deploy a fast, tested rollback path. Migrations must never block rolling back code.
- **Build Caching:** Cache dependencies and build layers in CI, keyed to lockfiles, for fast and reproducible builds.
- **Dependency Hell:** Commit lockfiles, pin versions, update in small regular batches, remove unused packages, and avoid new dependencies for trivial code.

## 13. Disaster recovery and incident response

- **Backups:** Automate encrypted backups stored apart from production (separate account or region), with point-in-time recovery for databases. Test restores on a schedule, because an untested backup does not count.
- **Disaster Recovery:** Define RTO (how long you can be down) and RPO (how much data you can lose), and keep a written, rehearsed recovery plan.
- **Multi-Region Deployment:** Use it only for strict availability or data residency requirements, since it adds major complexity in replication and consistency.
- **Chaos Engineering:** At maturity, inject failures deliberately (kill instances, add latency) to verify resilience assumptions. Start in staging.
- **Production Incidents:** Mitigate first (roll back, turn off a flag, scale up), find the root cause after, and keep users informed through a status page.
- **On-Call:** Maintain a clear rotation and escalation path, with alerts few and actionable enough to be sustainable.
- **Runbooks:** Write step-by-step guides for known failure modes and link them from alerts.
- **Postmortems:** Write blameless reviews with timeline, impact, root cause, and tracked action items. When fixing a production bug here, suggest what would prevent a recurrence.

## 14. Client side

- **Client Secrets:** Anything shipped to a browser or app is public. Variables with public prefixes (`NEXT_PUBLIC_`, `VITE_`, `EXPO_PUBLIC_`) end up in the bundle, so never put private keys in them.
- **Loading and Error States:** Give every async view loading, empty, and error states. Failures show a helpful message, never a blank screen.
- **Accessibility:** Use semantic HTML, labeled inputs, keyboard navigation, visible focus, sufficient contrast, and alt text. Target WCAG 2.2 AA.
- **Frontend Performance:** Watch bundle size, lazy-load heavy code, optimize images, and track Core Web Vitals.

## 15. AI and LLM features (when the app calls models)

- **Prompt Injection:** Treat model output and any retrieved, uploaded, or user-supplied content as untrusted. Never let it trigger privileged actions (tool calls, data access, payments) without validation or user confirmation.
- **Output Validation:** Validate structured output against a schema, and handle refusals, malformed output, and truncation.
- **Usage Controls:** Set per-user rate limits and quotas, maximum token limits, timeouts, and spend alerts on model APIs.
- **Model Data Handling:** Keep secrets and unnecessary personal data out of prompts, and know the provider's data retention terms.
- **Evals:** Keep a small set of test prompts with expected behavior to catch regressions when prompts or models change.

## Review output format

When asked for a review, audit, or production readiness check, respond in this structure:

1. Context: the assumed stage, stack, and scope reviewed, in one or two lines.
2. Findings: a table with the columns Severity, Area, Finding, Evidence (file and line), and Fix, sorted by severity. Critical means exploitable or losing data now. High means a likely outage, breach, or corruption. Medium means a reliability or maintainability risk. Low means worth doing later.
3. Tier 0 status: one line confirming which non-negotiables you checked and the result.
4. Not applicable yet: entries skipped for this stage, each with the trigger that would make it relevant.
5. Not verified: anything you could not inspect from the repository.
6. Next steps: the top 3 fixes in priority order.

Be specific and evidence-based. No generic advice, and no padding with entries that passed.
