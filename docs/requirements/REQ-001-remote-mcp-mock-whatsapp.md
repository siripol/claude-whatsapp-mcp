---
id: REQ-001
title: Secured Remote MCP Server with Mock WhatsApp Provider
priority: high
requires: []
acceptance:
  - The solution exposes a remote MCP server implemented in Python using FastMCP and runs locally inside Docker on the developer's Mac
  - The MCP server uses the current recommended HTTP-based remote transport supported by the selected FastMCP/MCP SDK rather than stdio as the primary transport
  - MCP clients access the exposed remote MCP endpoint through HTTPS; plain HTTP may be used only inside the trusted local Docker/network boundary when TLS terminates at a local ingress or reverse proxy
  - The remote MCP endpoint requires authentication before protected MCP tools can be invoked
  - Authentication credentials and related security configuration are supplied through environment variables or protected local runtime configuration and are never hardcoded in application source code
  - Authorization is enforced server-side before send_message can invoke the configured WhatsApp provider
  - Unauthenticated or unauthorized requests do not reach MockWhatsAppProvider
  - Basic configurable rate limiting protects send_message from accidental loops or excessive invocation
  - Each send_message invocation produces an audit record containing at minimum a timestamp, correlation/request ID, tool name, provider, and outcome, plus authenticated client identity where available, without recording credentials, access tokens, authorization headers, or other secrets
  - The MCP server exposes a send_message tool that accepts recipient and message as required inputs
  - Calling send_message invokes the configured WhatsApp provider and returns a structured result containing status, provider, message_id, recipient, message, and timestamp
  - The PoC uses MockWhatsAppProvider only and makes no calls to Meta WhatsApp Cloud API, Twilio, whatsapp-web.js, Baileys, or any other messaging service
  - MockWhatsAppProvider simulates successful delivery without sending a real message or incurring messaging cost
  - Message delivery is abstracted behind a provider interface so MockWhatsAppProvider can later be replaced by MetaWhatsAppProvider without changing the MCP tool contract wherever practical
  - The secured MCP endpoint is reachable from the host Mac through a documented local HTTPS endpoint after running docker compose up --build
  - Runtime configuration is supplied through environment variables and environment-specific values are not hardcoded in application source code
  - Secrets, certificates/private keys where applicable, and local environment files are excluded from Git and are never committed to the repository
  - Empty or whitespace-only recipient and message values are rejected and are not passed to the provider
  - Expected authentication, authorization, rate-limit, validation, and provider failures are returned as controlled errors and do not terminate the MCP server
  - Basic operational logging records server startup, request correlation ID, tool invocation, selected provider, delivery result, and controlled failures without exposing credentials or secrets
  - Automated tests cover MockWhatsAppProvider, send_message behavior, authentication, authorization, rate limiting, audit behavior, validation failure, and controlled provider failure
  - The repository contains the Docker artifacts required to build and run the complete secured PoC on macOS with Docker Desktop
  - The README documents prerequisites, architecture, local Docker setup, HTTPS endpoint, authentication setup, authorization behavior, rate limiting, audit behavior, MCP tool contract, test execution, and an example invocation
  - A clean clone can be configured from documented example configuration and started without modifying application source code
eval_threshold: 70
---

## Context

This proof of concept establishes the foundation of a secured remote Model Context Protocol (MCP) server that exposes outbound WhatsApp-style messaging to MCP-compatible AI clients.

The MCP server is not deployed to a public cloud in REQ-001. The complete runtime remains on the developer's Mac using Docker Desktop.

However, clients interact with it using a remote MCP transport and a security boundary that resembles a remotely deployed service:

```text
MCP Client
    |
    | HTTPS + Authentication
    v
Remote MCP Endpoint
    |
    | Authorization / Rate Limit / Audit
    v
MCP Server
```

The first phase intentionally does not connect to a real WhatsApp service.

A mock provider is used so MCP transport, HTTPS, authentication, authorization, rate limiting, audit, tool contracts, validation, provider abstraction, logging, Docker runtime, error handling, and tests can be implemented without a Meta Business account, WhatsApp phone number, or messaging cost.

The design must preserve clean extension points for:

- REQ-002 — Official Meta WhatsApp Provider
- REQ-003 — Production Remote Deployment and Security Hardening

## What

Build a secured, containerized Python Remote MCP Server exposing one outbound messaging tool:

```text
send_message(recipient: string, message: string)
```

The logical request flow is:

```text
MCP Client
    |
    | HTTPS
    | Authentication
    v
Remote MCP Endpoint
    |
    | Authorization
    | Rate Limiting
    | Audit
    v
MCP Server
    |
    | send_message
    v
WhatsAppProvider
    |
    v
MockWhatsAppProvider
    |
    v
Simulated Delivery Result
```

The server and supporting security components run locally in Docker on the developer's Mac.

No real WhatsApp message is transmitted.

## Architecture

### Logical Architecture

```text
MCP Client
    |
    | HTTPS + Authentication
    v
Remote MCP Endpoint
    |
    | Authorization
    | Rate Limit
    | Audit
    v
MCP Server
    |
    | send_message
    v
WhatsAppProvider
    |
    v
MockWhatsAppProvider
    |
    v
Simulated Delivery Result
```

### Local Runtime Architecture

```text
Developer Mac
|
+-- Docker Desktop
      |
      +-- Secured Remote MCP Runtime
            |
            +-- HTTPS Endpoint / TLS Termination
            +-- Authentication
            +-- Authorization
            +-- Rate Limiting
            +-- Audit
            +-- MCP Server
            |     |
            |     +-- send_message
            |
            +-- WhatsAppProvider
                  |
                  +-- MockWhatsAppProvider
```

The exact container boundaries are design decisions.

HTTPS termination, security middleware, and the MCP server may run in one container or in separate local containers where that produces a cleaner design.

REQ-001 does not require public Internet exposure.

## Functional Requirements

### MCP Tool

The server exposes:

```text
send_message
```

Inputs:

- `recipient`: required string
- `message`: required string

Both values must reject empty and whitespace-only input.

The MCP layer delegates message delivery to a provider abstraction rather than implementing WhatsApp-specific behavior directly.

A successful result contains at minimum:

- `status`
- `provider`
- `message_id`
- `recipient`
- `message`
- `timestamp`

Example:

```json
{
  "status": "success",
  "provider": "mock",
  "message_id": "mock-550e8400-e29b-41d4-a716-446655440000",
  "recipient": "+66812345678",
  "message": "Server CPU usage exceeded 90%",
  "timestamp": "2026-10-06T14:30:00Z"
}
```

For REQ-001, strict WhatsApp/E.164 telephone-number validation is optional unless selected during design.

Empty and whitespace-only validation is mandatory.

## Provider Abstraction

The design separates the MCP-facing tool from message-delivery implementation.

Conceptually:

```text
WhatsAppProvider
    |
    +-- MockWhatsAppProvider
    |       [REQ-001]
    |
    +-- MetaWhatsAppProvider
            [REQ-002 - Future]
```

The provider contract must support behavior equivalent to:

```text
send_message(recipient, message) -> MessageResult
```

The MCP tool must not depend directly on `MockWhatsAppProvider`.

Provider selection must be configuration-driven so a future provider can be introduced without rewriting the MCP-facing tool.

The provider abstraction must define a stable structured result and controlled provider-error behavior. Details that are specific to the future Meta implementation must not leak into the REQ-001 MCP tool contract.

## Mock WhatsApp Provider

`MockWhatsAppProvider` must:

- accept recipient and message
- generate a unique mock message ID
- generate a UTC timestamp
- return a successful structured result
- perform no external messaging or network call
- conform to the provider contract intended for the future Meta implementation

The mock provider must not require:

- Meta account
- WhatsApp Business Account
- WhatsApp phone number
- Meta access token
- message-template approval
- payment method
- Twilio account
- QR-code authentication

The PoC must therefore be runnable with zero WhatsApp messaging cost.

## MCP Transport

The primary server transport is remote HTTP-based MCP, not stdio.

Implementation must use the current recommended HTTP-based remote transport of the selected FastMCP/MCP SDK.

The exact MCP endpoint path may follow the selected SDK convention and must be documented.

The implementation must verify the current SDK behavior at development time rather than assuming a legacy SSE or HTTP convention.

The MCP application may listen internally on plain HTTP when it is protected behind a trusted local TLS termination boundary.

MCP clients must use the exposed HTTPS endpoint.

## Local Remote MCP Security

Although REQ-001 runs locally on the developer's Mac, access must follow the basic trust boundaries expected from a remote MCP service.

The secured request path is:

```text
MCP Client
    |
    | HTTPS + Authentication
    v
Remote MCP Endpoint
    |
    | Authorization
    | Rate Limit
    | Audit
    v
MCP Server
```

Security enforcement must be server-side and must not depend on instructions in an LLM prompt.

### HTTPS

MCP clients must connect through HTTPS.

TLS may terminate at:

- the MCP application
- a local reverse proxy
- a local ingress/proxy container
- another suitable Docker-hosted component selected during design

The exact implementation is a design decision.

Plain HTTP may be used only within the trusted local Docker/network boundary when TLS terminates before the MCP application.

Local certificates and private keys must not be committed to Git.

The README must document how the local development certificate and trust setup work.

### Authentication

An MCP client must authenticate before invoking protected MCP capabilities.

Unauthenticated requests must not be able to invoke `send_message`.

The authentication mechanism is selected during technical design based on:

- the current MCP specification
- FastMCP capabilities
- compatibility with the intended MCP client

Authentication credentials must:

- be externally configured
- never be hardcoded
- never appear in logs
- never be committed to Git

### Authorization

Authentication and authorization are separate controls.

After authentication, the server must verify that the authenticated caller is authorized to invoke `send_message`.

Authorization must be enforced before provider invocation.

An unauthorized request must not reach `MockWhatsAppProvider`.

The initial authorization model may be simple because REQ-001 is a local PoC, but its boundary and behavior must be explicit and testable.

### Rate Limiting

`send_message` must have basic rate-limit protection.

The purpose is to reduce the risk of:

- accidental invocation loops
- runaway MCP clients
- excessive tool calls
- uncontrolled future WhatsApp delivery when REQ-002 is introduced

Rate-limit configuration should be externalized where practical.

A rate-limited request must be rejected before invoking `MockWhatsAppProvider`.

### Audit

Each `send_message` invocation must produce an audit record sufficient to trace the request.

The audit information includes at minimum:

- timestamp
- correlation/request ID
- tool name
- provider
- outcome
- authenticated client identity where the selected authentication mechanism makes it available

Audit records must never contain:

- passwords
- access tokens
- authorization headers
- private keys
- future Meta credentials

Message-body logging should be disabled or redacted by default unless explicitly enabled for local debugging.

## Docker and Local Deployment

REQ-001 is **not** deployed to a public cloud.

The MCP server and any supporting security components run locally using Docker Desktop on the developer's Mac.

Expected runtime:

```text
Mac
 |
 +-- Docker Desktop
       |
       +-- Secured Remote MCP Runtime
              |
              +-- HTTPS Endpoint
              +-- Authentication
              +-- Authorization
              +-- Rate Limiting
              +-- Audit
              +-- MCP Server
              +-- MockWhatsAppProvider
```

The normal developer startup workflow must support:

```sh
docker compose up --build
```

After startup, the MCP client connects to the HTTPS endpoint exposed by the local Docker environment.

The containerized application must listen on interfaces required for Docker networking and port forwarding.

Stopping the Compose stack must stop the MCP runtime cleanly.

After the image is built, the application must not depend on Python packages installed globally on the host Mac.

## Configuration

Runtime and security configuration must be externalized.

Expected configuration includes values equivalent to:

- MCP host
- MCP port
- provider selection
- log level
- authentication configuration
- authorization configuration
- rate-limit configuration
- TLS/certificate configuration where applicable

The repository must provide safe example configuration.

Real local configuration, credentials, generated certificates/private keys where sensitive, and secrets must be excluded from Git.

Future Meta configuration may eventually include:

- WhatsApp access token
- WhatsApp phone-number ID
- template configuration

Those Meta values are not required or used by REQ-001.

## Logging

The application must provide operational logging sufficient to troubleshoot the local secured MCP service.

Logs should make it possible to determine:

- server startup and shutdown
- request/correlation ID
- authentication result without exposing credentials
- authorization result
- rate-limit rejection
- MCP tool invocation
- selected provider
- generated message ID
- delivery result
- controlled errors

Credentials, tokens, authorization headers, private keys, and future Meta secrets must never be logged.

Message content should be redacted or omitted by default where practical.

Operational logs and audit records may share an output mechanism for the PoC, but audit fields must remain identifiable and testable.

## Error Handling

Expected failures must not terminate the MCP server.

Controlled failure categories include:

- unauthenticated request
- unauthorized request
- rate-limit rejection
- empty recipient
- empty message
- provider failure
- invalid runtime configuration

The exact MCP error representation may follow the selected SDK, but failures must remain observable and controlled.

Security failures must not expose implementation details, credentials, or internal stack traces to the client.

Validation, authentication, authorization, and rate-limit failures must be rejected before provider invocation.

## Testing

Automated tests must cover at minimum:

1. Successful mock delivery.
2. Unique message ID generation.
3. Recipient preservation.
4. Message preservation.
5. Timestamp generation.
6. Empty recipient rejection.
7. Empty message rejection.
8. Controlled provider failure.
9. Unauthenticated request rejection.
10. Unauthorized request rejection.
11. Verification that unauthorized requests do not invoke `MockWhatsAppProvider`.
12. Rate-limit enforcement.
13. Verification that rate-limited requests do not invoke `MockWhatsAppProvider`.
14. Audit and correlation/request ID generation.
15. Secret and credential redaction from logs and audit output.
16. Secured MCP tool invocation through the local Docker runtime.

Normal automated tests must not call any external messaging service.

Where HTTPS integration tests require local development certificates, setup must be deterministic and documented.

Tests must be runnable from a clean clone using documented commands and configuration.

## Repository and Documentation

REQ-001 does not prescribe the final `src/` or `tests/` layout before technical design.

The implementation structure must be derived from the requirement and approved design.

At minimum, the completed implementation is expected to contain appropriate artifacts for:

- Python application/package
- MCP server
- MCP tool
- provider abstraction
- `MockWhatsAppProvider`
- security controls
- configuration
- automated tests
- Docker image
- Docker Compose runtime
- example environment/configuration
- README

Requirements remain under:

```text
docs/requirements/
```

Technical design documentation is produced separately under:

```text
docs/design/
```

The README must document:

- prerequisites for macOS and Docker Desktop
- architecture and trust boundaries
- local configuration and secret handling
- local certificate generation and trust setup
- build, startup, shutdown, and restart commands
- the exposed HTTPS MCP endpoint and transport
- authentication and authorization setup
- rate-limit behavior and configuration
- audit behavior
- the `send_message` input/output contract
- an example authenticated invocation
- test commands
- known local-PoC limitations

## Out of Scope

The following are explicitly outside REQ-001:

- Real WhatsApp message delivery
- Meta WhatsApp Cloud API integration
- Meta Business verification
- WhatsApp Business Account configuration
- Production WhatsApp phone-number registration
- WhatsApp message-template creation or approval
- WhatsApp pricing or billing
- Twilio integration
- `whatsapp-web.js`
- Baileys
- QR-code-based WhatsApp authentication
- Reading incoming WhatsApp messages
- Listing WhatsApp conversations
- Reading message history
- Sending images, documents, audio, or video
- Two-way WhatsApp chatbot functionality
- Scheduled alerts
- LLM-generated alert content
- RAG
- Multi-agent workflows
- Public Internet exposure
- Public cloud hosting
- Production secret-management platform
- High availability
- Horizontal scaling
- Multi-region deployment
- Production monitoring/alerting platform

HTTPS, authentication, authorization, basic rate limiting, and audit are **not** out of scope. They are mandatory local-PoC capabilities in REQ-001.

## Relationship to Future Requirements

### REQ-002 — Official Meta WhatsApp Provider

REQ-002 replaces or supplements `MockWhatsAppProvider` with `MetaWhatsAppProvider` using the official Meta WhatsApp Cloud API.

The MCP-facing `send_message` contract should remain stable wherever practical.

Target extension:

```text
MCP Server
    |
    v
WhatsAppProvider
    |
    +-- MockWhatsAppProvider
    |
    +-- MetaWhatsAppProvider
              |
              v
    Meta WhatsApp Cloud API
```

REQ-002 owns real provider credentials, Meta-specific request/response mapping, WhatsApp delivery behavior, and Meta-specific error handling. It must preserve the server-side security boundaries established by REQ-001.

### REQ-003 — Production Remote Deployment and Security Hardening

REQ-003 takes the security boundaries proven locally in REQ-001 and moves the service to a remotely hosted production-oriented environment.

REQ-003 focuses on:

- public remote hosting
- production TLS/certificate management
- production-grade identity/authentication
- stronger authorization policy
- production secret management
- production monitoring and observability
- hardened container/runtime configuration
- operational recovery
- deployment and rollback
- production incident procedures

REQ-003 must not assume that REQ-001's local development security implementation is automatically sufficient for production.

The intended progression is:

```text
REQ-001
Local Docker on Mac
+ Remote MCP
+ HTTPS
+ Authentication
+ Authorization
+ Rate Limit
+ Audit
+ Mock WhatsApp
        |
        v
REQ-002
Official Meta WhatsApp Provider
        |
        v
REQ-003
Public/Cloud Deployment
+ Production Secret Management
+ Production TLS
+ Monitoring
+ Hardening
+ Recovery and Operations
```

## Definition of Done

REQ-001 is complete when a developer can:

1. Clone the repository.
2. Configure the application using documented local configuration without modifying source code.
3. Build and start the complete runtime using Docker Compose on a Mac.
4. Connect an MCP-compatible client to the Docker-hosted MCP endpoint using HTTPS.
5. Demonstrate that an unauthenticated client cannot invoke `send_message`.
6. Authenticate successfully.
7. Demonstrate server-side authorization before provider invocation.
8. Invoke `send_message` as an authorized client.
9. Observe rate-limit protection.
10. Observe an audit record with a correlation/request ID and no exposed secret.
11. Observe `MockWhatsAppProvider` process the authorized request.
12. Receive a structured successful mock delivery response.
13. Confirm that no external WhatsApp service was contacted.
14. Run the automated tests successfully.
15. Stop and restart the Docker runtime without modifying application source code.

REQ-001 must not be marked complete while any mandatory acceptance criterion fails.

## Open Questions

The following questions must be resolved during technical design or implementation planning. They are not permission to weaken the mandatory acceptance criteria.

1. Which exact Python MCP/FastMCP package and pinned version will be used?
2. What is the current recommended HTTP-based remote MCP transport for that version, and what endpoint path does it expose?
3. Which target MCP client or clients must be supported for the PoC?
4. Which authentication mechanism best fits the current MCP specification, selected FastMCP version, and target client?
5. What client identity or claim will be used for authorization, and what minimal policy grants access to `send_message`?
6. Where will TLS terminate: in the MCP application or in a local reverse-proxy/ingress container?
7. How will local development certificates be generated and trusted on macOS while ensuring private keys remain uncommitted?
8. What default rate limit, time window, and identity key should be used, and is in-memory state sufficient for this single-host PoC?
9. Where will audit records be written, in what structured format, and what retention behavior is appropriate for local development?
10. Should recipient validation remain limited to non-empty input or enforce E.164 formatting in REQ-001?
11. Should message bodies always be omitted/redacted from operational logs and audit records, or may an explicit local-debug option enable them?
12. Should HTTPS/security and MCP server responsibilities run in one container or separate Compose services?
13. Which error codes and structured error fields are supported by the selected MCP SDK for authentication, authorization, rate-limit, validation, and provider failures?
14. What deterministic strategy will integration tests use for local certificates, authentication credentials, authorization identities, rate-limit state, and audit capture?

All resolved answers must be reflected consistently in the technical design, implementation, automated tests, example configuration, and README.
