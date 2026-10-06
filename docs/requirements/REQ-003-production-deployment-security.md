---
id: REQ-003
title: Production Remote Deployment and Security Hardening
priority: high
requires: [REQ-001, REQ-002]
acceptance:
  - The completed REQ-001 and REQ-002 solution is deployed to an approved remotely hosted production-oriented environment without breaking the MCP-facing send_message contract wherever practical
  - MCP clients reach the production remote MCP endpoint only through HTTPS using a valid trusted certificate and supported modern TLS configuration
  - Plain HTTP is not publicly exposed and may exist only inside explicitly trusted private network boundaries where TLS termination is documented
  - Production authentication is integrated with an approved identity mechanism compatible with the selected MCP clients and current MCP specification
  - Production authorization is enforced server-side using an explicit least-privilege policy before send_message can invoke any provider
  - Unauthenticated, unauthorized, invalid, or rate-limited requests do not invoke MetaWhatsAppProvider or any external messaging provider
  - Production credentials, Meta tokens, private keys, and other secrets are obtained from an approved secret-management mechanism and are not stored in source code, container images, deployment manifests, or plaintext repository files
  - Production rate limiting and abuse controls protect the public MCP endpoint and the send_message tool using an identity-aware or otherwise documented policy
  - Each protected request and send_message invocation produces correlated security and audit records sufficient for incident investigation without recording credentials, authorization headers, private keys, or message bodies by default
  - Centralized operational logs, metrics, health signals, and alerts provide visibility into availability, latency, authentication failures, authorization failures, rate-limit events, provider outcomes, and controlled errors
  - The production workload runs with documented container and runtime hardening, least privilege, restricted network access, and no unnecessary public services
  - Deployment is reproducible through version-controlled deployment configuration or infrastructure automation that contains no secrets
  - A documented rollback procedure can restore the last known good application and configuration version
  - Health checks, graceful shutdown, restart behavior, timeout behavior, and dependency failure handling are implemented and verified
  - The service preserves REQ-001 correlation, audit, provider abstraction, validation, and controlled-error behavior and preserves REQ-002 official Meta integration
  - Production delivery failures, upstream Meta throttling, dependency outages, and transient network failures are observable and do not crash the MCP service
  - Security-sensitive logs and audit records have documented access controls, retention, redaction, and disposal behavior
  - Automated tests cover production security configuration, authorization boundaries, secret redaction, rate limiting, deployment configuration, health behavior, and representative failure paths without sending uncontrolled real messages
  - A staging or equivalent pre-production verification path validates the deployable artifact and critical end-to-end flow before production promotion
  - Production readiness documentation includes architecture, trust boundaries, deployment, rollback, secrets, certificate lifecycle, monitoring, alerting, incident response, recovery, and operational ownership
  - The public deployment does not expose debug endpoints, development credentials, local certificates, stack traces, or unnecessary administrative interfaces
  - Production readiness is approved only after unresolved risk decisions, capacity targets, recovery objectives, data-retention rules, and operational ownership are documented
eval_threshold: 70
---

## Context

REQ-001 creates a secured remote MCP proof of concept running locally in Docker Desktop. It establishes the MCP endpoint, `send_message` contract, provider abstraction, HTTPS, authentication, authorization, basic rate limiting, audit records, logging, configuration, and tests.

REQ-002 adds `MetaWhatsAppProvider` and real outbound WhatsApp delivery through the official Meta WhatsApp Cloud API while retaining the local Docker runtime and REQ-001 security boundaries.

REQ-003 moves the resulting system into an approved remotely hosted, production-oriented environment and hardens the entire request path for public or controlled remote use.

REQ-003 does not merely add the same security controls already required by REQ-001. It replaces local-development implementations and assumptions where necessary with production-grade identity, secrets, TLS, runtime protection, observability, recovery, deployment, and operational controls.

Passing REQ-001 locally does not prove production readiness. Every inherited control must be reviewed against the selected production platform, threat model, client population, data sensitivity, and operational requirements.

## What

Deploy and operate the secured MCP and Meta WhatsApp integration as a production-oriented remote service.

The target request flow is:

```text
MCP Client
    |
    | Public/Remote HTTPS
    | Production Authentication
    v
Production MCP Endpoint
    |
    | Authorization
    | Rate Limit / Abuse Protection
    | Correlation / Audit
    v
MCP Service
    |
    | send_message
    v
WhatsAppProvider
    |
    v
MetaWhatsAppProvider
    |
    | Outbound HTTPS
    v
Official Meta WhatsApp Cloud API
```

Supporting production capabilities include:

- trusted TLS and certificate lifecycle
- production identity and authorization
- managed secret handling
- hardened network and runtime boundaries
- centralized logging, metrics, alerts, and audit access
- repeatable deployment and rollback
- health management and graceful shutdown
- incident response and recovery documentation
- staging or equivalent pre-production validation

The exact hosting vendor and service topology are design decisions unless separately constrained.

## Architecture

### Logical Production Architecture

```text
Remote MCP Clients
        |
        | HTTPS
        v
Public DNS / Trusted TLS Boundary
        |
        v
Ingress / Gateway / Edge Protection
        |
        | Authentication
        | Coarse abuse protection
        v
Authorization / Tool Rate Limit / Audit
        |
        v
MCP Application Runtime
        |
        +-- send_message
        +-- WhatsAppProvider
        +-- MetaWhatsAppProvider
        |
        | outbound HTTPS only
        v
Meta WhatsApp Cloud API

Supporting services:
  - Identity and access management
  - Secret management
  - Central logging and audit storage
  - Metrics and alerting
  - Deployment/configuration management
```

The exact boundary between ingress, authentication, application authorization, rate limiting, and audit is selected during technical design. Regardless of placement, enforcement must be testable end to end.

### Trust Boundaries

The design must explicitly document at minimum:

1. The untrusted client-to-public-endpoint boundary.
2. The TLS termination and ingress boundary.
3. The authenticated identity-to-authorization boundary.
4. The application-to-secret-management boundary.
5. The application-to-Meta external API boundary.
6. The workload-to-observability boundary.
7. The operator-to-production-control-plane boundary.

Traffic and identity assumptions at each boundary must be explicit. Network location alone must not be treated as authorization to invoke `send_message`.

## Functional Continuity

REQ-003 must preserve the functional behavior delivered by REQ-001 and REQ-002.

The MCP-facing tool remains compatible wherever practical:

```text
send_message(recipient: string, message: string)
```

A successful response remains provider-neutral and includes at minimum:

- `status`
- `provider`
- `message_id`
- `recipient`
- `message`
- `timestamp`

Production deployment must not introduce a bypass around validation, authentication, authorization, rate limiting, audit, or provider abstraction.

Any necessary contract change must be documented, versioned, tested for compatibility, and explicitly approved. Infrastructure convenience alone is not sufficient justification for a breaking MCP tool change.

## Production Hosting and Deployment

The service must run in an approved remote hosting environment suitable for the agreed production risk and availability targets.

The selected deployment model may use managed containers, an orchestrated container platform, virtual machines, or another approved runtime. The technical design must explain why the selected model satisfies the requirements.

Production deployment must provide:

- a stable remote HTTPS endpoint
- repeatable environment provisioning or deployment configuration
- immutable or uniquely versioned application artifacts
- separation of production and non-production configuration
- safe secret injection at runtime
- controlled promotion from pre-production to production
- version and deployment traceability
- documented rollback
- health checks and restart behavior
- graceful termination for in-flight requests where supported

Deployment configuration and infrastructure definitions must be version-controlled without embedding credentials or private keys.

Manual production changes must be minimized, authorized, and documented. Emergency changes must leave an auditable record and be reconciled into the maintained configuration afterward.

## Network Security

Only required public services and ports may be exposed.

The production network design must:

- expose the MCP endpoint through the approved HTTPS boundary
- prevent direct public access to internal application ports where a gateway or ingress is used
- restrict administrative and control-plane access
- restrict outbound network access to required destinations where practical
- allow access to approved identity, secret, observability, and Meta API dependencies
- document trusted proxy behavior and client-address handling
- prevent spoofed forwarding headers from bypassing identity or rate-limit controls

Private network placement may reduce exposure but must not replace authentication and authorization.

Debug, metrics, health, or administrative endpoints must be separately protected and must not disclose secrets or unnecessary implementation detail.

## TLS and Certificate Management

Production MCP clients must connect through HTTPS using a certificate that is valid for the production hostname and trusted by intended clients.

The design must define:

- TLS termination location
- supported TLS policy
- certificate issuer and trust model
- automated or operational issuance
- renewal and rotation
- private-key storage and access
- expiration monitoring
- failure and recovery procedure

Plain HTTP must not be publicly exposed. If HTTP exists between trusted internal components after TLS termination, the boundary, network controls, risk, and any plan for internal encryption must be documented.

Development certificates from REQ-001 must not be reused in production.

## Production Authentication

Production authentication must use an approved mechanism compatible with:

- the current MCP specification
- the selected MCP server/FastMCP version
- intended production MCP clients
- the selected identity provider or credential authority
- token or credential lifecycle requirements

The design must address:

- credential issuance
- identity claims
- validation and trust roots
- expiration
- revocation
- rotation
- audience and scope validation where applicable
- failure behavior
- operator access separation

Static local-development credentials from REQ-001 must not be promoted to production unless explicitly risk-assessed and approved. Authentication secrets must not be embedded in clients, source code, images, or public configuration.

Authentication failures must be controlled, auditable, and free of sensitive diagnostic detail.

## Production Authorization

Authentication and authorization remain separate controls.

The server must enforce an explicit least-privilege authorization policy before `send_message` reaches any provider.

The policy must define:

- which authenticated identities may invoke `send_message`
- which claims, roles, scopes, or attributes are trusted
- default-deny behavior
- how policy changes are reviewed and deployed
- how authorization decisions are logged and audited
- how revoked access takes effect

If recipient or account restrictions are required, they must be enforced server-side and included in the policy or provider-neutral validation layer.

The gateway may perform preliminary checks, but application-side or equivalently authoritative authorization must prevent a gateway misconfiguration from silently allowing provider invocation.

## Secret Management

Production secrets must be provided by an approved secret-management mechanism.

Protected values include at minimum:

- Meta access tokens
- MCP client credential material where applicable
- identity-provider client secrets
- TLS private keys where managed by the workload
- signing or encryption keys
- credentials for logging, audit, metrics, or deployment systems

Secrets must not be stored in:

- application source code
- Git history
- container images
- plaintext deployment manifests
- documentation examples
- build logs
- unprotected environment exports or support bundles

The design must define secret access, least privilege, rotation, revocation, expiration, break-glass handling, and auditability.

The service must fail safely when required secrets are missing or invalid. Client errors and logs must not reveal secret values.

## Rate Limiting and Abuse Protection

Production controls must protect both the public endpoint and the cost-bearing `send_message` operation.

The design must address:

- identity-aware limits where reliable client identity is available
- endpoint-level and tool-level limits
- burst and sustained limits
- distributed state if multiple application instances are used
- behavior when the rate-limit dependency is unavailable
- client feedback and retry guidance
- alerting for unusual rejection volume
- protection from accidental loops and deliberate abuse
- interaction with Meta upstream limits

A rate-limited request must be rejected before invoking `MetaWhatsAppProvider`.

The final thresholds are operational configuration and must be derived from agreed usage and capacity targets. They must not be hardcoded in application source code.

## Audit, Logging, and Correlation

Every protected request must receive or propagate a correlation/request ID.

Each `send_message` attempt must produce an audit record containing at minimum:

- timestamp
- correlation/request ID
- authenticated identity or stable identity reference
- authorization outcome
- tool name
- provider
- request outcome
- controlled failure category where applicable
- provider message ID on success where permitted

Audit records must distinguish:

- rejection before application processing
- authentication failure
- authorization failure
- validation failure
- rate-limit rejection
- provider invocation
- provider acceptance
- provider failure

Logs and audit records must not contain credentials, access tokens, authorization headers, private keys, or raw secret-management values.

Message bodies must be omitted or redacted by default. Recipient values must be masked, tokenized, or otherwise handled according to the approved data policy.

The design must define access control, storage, integrity expectations, retention, search, export, and disposal for logs and audit records.

## Monitoring, Metrics, and Alerting

The production service must expose or emit signals sufficient to determine health and detect material failures.

Operational visibility must cover at minimum:

- deployment version and startup status
- instance or workload health
- request volume
- request latency
- success and controlled-error counts
- authentication and authorization failures
- rate-limit rejections
- provider request latency and outcomes
- timeout and dependency failure rates
- certificate expiration risk
- secret/configuration retrieval failures
- restart or crash behavior

Alerts must be actionable and must identify an owner or response path.

Alert thresholds, paging policy, dashboards, and service objectives are defined during production design based on agreed operational targets.

Observability data must follow the same secret-redaction and data-handling rules as application logs.

## Runtime and Container Hardening

The production workload must follow least-privilege runtime practices appropriate to the selected platform.

The design must address:

- non-root execution where supported
- minimal production image and dependencies
- pinned and reviewable dependency versions
- vulnerability scanning and remediation workflow
- read-only filesystem where practical
- restricted Linux capabilities and privilege escalation
- controlled writable paths
- resource requests and limits where the platform supports them
- secure temporary storage
- restricted service-account permissions
- separation of build and runtime credentials
- signed or traceable build artifacts where practical
- protection of debug and administrative features

No shell, package manager, or development tool should be included in the runtime image unless required and justified.

Platform-specific controls must be documented in the technical design rather than assumed.

## Reliability, Health, and Dependency Failure

The service must handle expected dependency failures without uncontrolled termination or secret leakage.

The design must include:

- startup/readiness/liveness behavior appropriate to the platform
- graceful shutdown
- finite outbound timeouts
- bounded concurrency and resource use
- controlled Meta API failure mapping
- behavior during identity, secret, audit, rate-limit, or observability dependency failures
- restart policy
- safe deployment and rollback

The system must not report readiness before mandatory configuration and dependencies required for safe request handling are available.

Retries must be bounded and must not create an unreasonable duplicate-message risk. Retry behavior must remain consistent with REQ-002 delivery-safety requirements.

High availability, multi-region deployment, and disaster-recovery topology depend on agreed service objectives and remain open design decisions unless explicitly approved.

## Data Protection and Privacy

The production design must identify data processed or produced by the service, including:

- client identity
- recipient identifiers
- message content
- Meta message IDs
- timestamps
- IP or network metadata
- audit and operational records

For each category, the design must define or reference:

- purpose
- sensitivity
- storage locations
- encryption expectations
- access control
- retention
- deletion/disposal
- masking or redaction
- operator visibility

The service must minimize stored message content. Message bodies must not be persisted merely for debugging convenience unless explicitly approved with a retention and access policy.

Applicable legal, privacy, data-residency, and organizational requirements must be identified before production approval.

## Deployment, Promotion, and Rollback

Production deployment must use a repeatable, reviewable process.

The process must provide:

- a uniquely identifiable application artifact
- automated or documented configuration validation
- pre-production verification
- security and automated test gates
- production approval appropriate to the organization
- deployment status and history
- post-deployment smoke verification
- rollback criteria
- rollback procedure
- confirmation after rollback

The deployable artifact tested in staging or an equivalent environment should be the artifact promoted to production, with environment-specific configuration supplied externally.

Database migration behavior is not required unless the approved design introduces persistent state. If persistent state is introduced, migration, backup, restoration, and rollback compatibility must be specified before production use.

## Recovery and Incident Response

Production documentation must cover:

- service outage triage
- suspected credential compromise
- Meta token revocation or expiration
- certificate failure or expiration
- identity-provider failure
- excessive or abusive message invocation
- unexpected delivery or duplicate-message incident
- audit/logging outage
- deployment rollback
- secret rotation
- restoration of required configuration or persistent state
- evidence preservation and escalation

Recovery procedures must identify required roles and access. Break-glass access must be limited, auditable, and revoked or rotated after use.

Recovery time and recovery point objectives must be agreed before production approval if the service stores persistent operational or audit state.

## Configuration

Production configuration must be externalized and separated from application code.

Expected configuration includes values equivalent to:

- public hostname and MCP endpoint
- runtime environment
- provider selection
- Meta API configuration references
- identity and authorization configuration
- TLS and trusted-proxy configuration
- endpoint and tool rate limits
- request timeouts and concurrency limits
- log level and redaction policy
- audit destination and retention settings
- metrics and alert configuration
- health-check behavior

Configuration must be validated before the service reports readiness.

Unsafe defaults, placeholder credentials, development certificates, debug mode, and permissive authorization must not be enabled in production.

## Error Handling

Client-visible production errors must:

- follow the selected MCP SDK's supported error model
- remain controlled and machine-identifiable where practical
- avoid internal stack traces and platform details
- avoid secrets and raw upstream payloads
- preserve a correlation/request ID for support
- distinguish retryable from non-retryable conditions only where safe and useful

Internal diagnostics must contain enough structured context for operations while following redaction and privacy requirements.

Failure in logging or observability must not silently authorize a request. Failure in security-critical dependencies must follow the documented fail-open or fail-closed policy, with fail-closed required unless a different behavior is explicitly risk-approved.

## Testing and Verification

Normal tests must not send uncontrolled real messages.

Testing must cover at minimum:

1. Production configuration validation.
2. HTTPS-only external access and trusted certificate validation.
3. Authentication success, expiration, invalidity, and revocation behavior where testable.
4. Default-deny authorization and least-privilege tool access.
5. Verification that rejected requests never invoke the provider.
6. Endpoint and tool-level rate limiting.
7. Distributed rate-limit behavior if multiple instances are used.
8. Secret injection and secret-redaction behavior.
9. Audit completeness and correlation across the request path.
10. Log and recipient/message redaction.
11. Meta provider success and controlled failures using stubs in normal automation.
12. Health, readiness, graceful shutdown, and restart behavior.
13. Timeout and dependency-failure behavior.
14. Container/runtime security configuration.
15. Deployment configuration linting or validation.
16. Pre-production end-to-end MCP invocation.
17. Rollback rehearsal or verified rollback procedure.
18. Certificate renewal or rotation procedure where testable.
19. Secret rotation procedure where testable.
20. Representative alert generation and routing.

Security testing must include an approved combination of:

- dependency and container-image scanning
- static analysis
- secret scanning
- configuration and infrastructure scanning
- negative authorization tests
- public-surface review
- targeted penetration or adversarial testing proportional to risk

Any production smoke test that can send a real message must be explicitly controlled, use an approved test recipient, minimize cost, and avoid sending duplicate or unexpected messages.

## Repository and Documentation

The repository must contain or reference appropriate version-controlled artifacts for:

- production deployment configuration or infrastructure automation
- environment-safe application configuration
- build and release process
- security validation
- automated tests
- monitoring and alert definitions where managed as code
- dashboards where managed as code
- runbooks
- deployment and rollback
- incident response
- architecture and trust boundaries
- operational ownership

Secrets and private keys must never be committed.

Production documentation must include:

- system context and component architecture
- public and private network boundaries
- identity and authorization flow
- secret and certificate lifecycle
- configuration reference
- deployment and promotion procedure
- rollback procedure
- health and dependency behavior
- dashboards and alert response
- audit access and retention
- security hardening decisions
- known risks and accepted exceptions
- incident and recovery procedures
- contact or ownership model

Documentation must clearly distinguish production requirements, design decisions, platform-specific instructions, assumptions, and unresolved risks.

## Out of Scope

The following are outside REQ-003 unless separately approved:

- New MCP tools unrelated to `send_message`
- Replacing the official Meta provider with an unofficial provider
- Incoming WhatsApp chatbot functionality
- Reading conversations or message history
- Media, document, audio, video, commerce, or interactive-message features
- Bulk marketing campaigns
- Customer relationship management features
- LLM-generated message content
- RAG or multi-agent workflows
- End-user administration portal
- Billing platform implementation
- Guaranteed exactly-once delivery across Meta and recipient devices
- Multi-region active-active deployment without an approved service objective
- Unlimited horizontal scale
- Platform migration unrelated to the selected production design
- Formal compliance certification unless separately required
- Twenty-four-hour on-call staffing unless separately agreed

REQ-003 may create extension points or operational recommendations for future requirements but must not silently add product functionality.

## Relationship to Prior Requirements

### REQ-001 — Secured Remote MCP Server with Mock WhatsApp Provider

REQ-003 inherits and productionizes:

- remote HTTP-based MCP behavior
- the `send_message` contract
- provider abstraction
- validation and controlled errors
- HTTPS, authentication, authorization, rate limiting, audit, and correlation boundaries
- configuration and automated-test foundations

Local credentials, local certificates, single-process rate-limit assumptions, local logs, and developer-machine trust assumptions must be replaced or explicitly reassessed.

### REQ-002 — Official Meta WhatsApp Provider

REQ-003 deploys and operates the official Meta integration established by REQ-002.

It must preserve:

- `MetaWhatsAppProvider`
- provider-neutral results and errors
- safe provider selection
- Meta credential redaction
- timeout and delivery-safety behavior
- separation between normal stubbed tests and controlled real-message verification

Production changes should not require rewriting the provider or MCP contract wherever practical.

## Definition of Done

REQ-003 is complete when the approved production owner can:

1. Provision or identify the production environment through a documented and reviewable process.
2. Deploy a uniquely versioned artifact that passed automated and pre-production checks.
3. Expose the MCP endpoint through trusted HTTPS with no unintended public ports or debug interfaces.
4. Authenticate an approved production MCP client using the selected production identity mechanism.
5. Demonstrate default-deny server-side authorization and least-privilege access to `send_message`.
6. Demonstrate that rejected requests do not reach `MetaWhatsAppProvider`.
7. Retrieve required secrets through the approved production mechanism without embedding them in code, images, or manifests.
8. Invoke `send_message` through the full production path using an approved test recipient.
9. Trace the invocation through correlation, audit, logs, metrics, and provider outcome without exposing sensitive data.
10. Demonstrate production rate limiting and abuse controls before provider invocation.
11. Demonstrate health checks, controlled dependency failure, graceful shutdown, and restart behavior.
12. Observe actionable monitoring and at least one safely generated alert path.
13. Verify certificate and secret rotation procedures.
14. Execute or rehearse the documented rollback procedure and confirm service recovery.
15. Demonstrate that the normal automated suite does not send uncontrolled real messages.
16. Complete production-readiness review with operational ownership, known risks, exceptions, capacity targets, recovery objectives, and data-retention decisions documented.

REQ-003 must not be marked complete while any mandatory acceptance criterion fails or a critical production risk remains unowned.

## Open Questions

The following questions must be resolved during production technical design and readiness review:

1. Which cloud, hosting platform, region, and runtime model are approved?
2. What production hostname and DNS ownership model will be used?
3. Where will TLS terminate, who issues certificates, and how are renewal and expiration monitored?
4. Which production identity provider and MCP-compatible authentication flow will be used?
5. Which identities, roles, scopes, or attributes may invoke `send_message`?
6. Are recipient allowlists, tenant boundaries, or per-client sending restrictions required?
7. What endpoint and tool-level rate limits are appropriate for expected usage and cost risk?
8. Must rate-limit state work across multiple instances, regions, or deployments?
9. Which secret-management service and rotation schedule are approved?
10. Which logging, metrics, audit, dashboard, and alerting platforms will be used?
11. What data may be logged, how must recipient identifiers be masked, and what are the retention and deletion periods?
12. What legal, privacy, residency, or organizational policies apply to recipient data and message content?
13. What availability target, latency objective, capacity, concurrency, and cost budget are required?
14. Are high availability and horizontal scaling required for the first production release?
15. What are the recovery time and recovery point objectives?
16. What fail-open or fail-closed behavior is required when identity, authorization, rate-limit, audit, secret, or observability dependencies fail?
17. Which continuous-integration and deployment system will build, scan, promote, deploy, and roll back releases?
18. What staging or equivalent pre-production environment is available, and how closely must it match production?
19. Who owns deployment approval, operational response, security incidents, Meta account administration, and break-glass access?
20. What vulnerability-severity thresholds and remediation timelines block production release?
21. Is a formal penetration test, threat-model review, or compliance assessment required before launch?
22. Which production smoke test recipient and cost controls will prevent accidental or duplicate real messages?
23. Does any component introduce persistent state requiring backup, restoration, migration, or disaster-recovery procedures?
24. Which risks may be accepted for the first production release, who may accept them, and when must they be revisited?

All resolved answers must be reflected consistently in the production design, threat model, deployment artifacts, automated tests, runbooks, readiness review, and operational ownership records.
