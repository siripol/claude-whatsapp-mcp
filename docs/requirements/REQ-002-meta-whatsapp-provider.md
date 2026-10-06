---
id: REQ-002
title: Official Meta WhatsApp Provider
priority: high
requires: [REQ-001]
acceptance:
  - REQ-001 is complete and its secured remote MCP endpoint, send_message tool contract, provider abstraction, and local Docker runtime remain operational
  - The solution implements MetaWhatsAppProvider using the official Meta WhatsApp Cloud API and does not use Twilio, whatsapp-web.js, Baileys, browser automation, or QR-code authentication
  - MetaWhatsAppProvider conforms to the WhatsAppProvider contract established by REQ-001
  - Provider selection is configuration-driven and supports selecting mock or meta without changing the MCP-facing send_message implementation
  - The existing send_message input contract remains compatible with REQ-001 wherever practical
  - A successful Meta delivery request returns a structured result containing status, provider, message_id, recipient, message, and timestamp
  - A successful Meta result identifies the provider as meta and uses the message identifier returned by the official Meta API
  - The implementation validates required recipient and message inputs before invoking MetaWhatsAppProvider
  - The implementation documents and enforces the supported WhatsApp text or template delivery mode and its preconditions
  - Meta access tokens, phone-number IDs, business identifiers, and related credentials are externally configured and never hardcoded in application source code
  - Meta credentials, authorization headers, secrets, and sensitive response fields never appear in operational logs, audit records, client errors, or committed files
  - The HTTPS, authentication, authorization, rate limiting, correlation, and audit controls established in REQ-001 continue to protect send_message when the meta provider is selected
  - Unauthenticated, unauthorized, invalid, or locally rate-limited requests do not invoke the Meta WhatsApp Cloud API
  - Meta API errors, timeouts, authentication failures, permission failures, recipient failures, policy failures, and upstream rate limits are translated into controlled provider errors without terminating the MCP server
  - Automatic retry behavior does not create an unreasonable duplicate-message risk and is documented and tested if enabled
  - Normal automated tests use mocked or stubbed Meta API responses and do not send real WhatsApp messages or incur messaging cost
  - Any live Meta integration test is explicitly opt-in, disabled by default, clearly warns about real delivery and cost, and uses externally supplied credentials
  - Automated tests cover successful Meta response mapping, provider selection, request construction, credential redaction, representative Meta failures, timeout behavior, and preservation of REQ-001 security boundaries
  - The complete REQ-002 runtime remains runnable locally with Docker Desktop on the developer's Mac
  - The README documents Meta prerequisites, configuration, supported messaging mode, recipient requirements, local Docker execution, test strategy, expected costs or account limitations, and a real-message verification procedure
  - A developer can switch between mock and meta providers through documented configuration without modifying application source code
  - REQ-002 does not implement production public hosting or claim that local PoC controls are production-ready
eval_threshold: 70
---

## Context

REQ-001 establishes a secured remote MCP proof of concept running locally in Docker Desktop on the developer's Mac. It provides:

- an HTTPS remote MCP endpoint
- authentication and server-side authorization
- basic rate limiting and audit records
- the `send_message` MCP tool
- a `WhatsAppProvider` abstraction
- `MockWhatsAppProvider`
- configuration, logging, error handling, automated tests, and Docker artifacts

REQ-002 extends that foundation with real outbound WhatsApp delivery through the official Meta WhatsApp Cloud API.

REQ-002 must not rebuild or bypass the MCP and security boundaries from REQ-001. Its primary responsibility is to add `MetaWhatsAppProvider`, integrate it through the existing provider abstraction, and verify real delivery in a controlled way.

REQ-002 continues to run locally in Docker on the developer's Mac. Public or cloud production deployment belongs to REQ-003.

## What

Implement an official Meta-backed provider for the existing MCP tool:

```text
send_message(recipient: string, message: string)
```

The target request flow is:

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
    |
    | send_message
    v
WhatsAppProvider
    |
    +-- MockWhatsAppProvider   [REQ-001]
    |
    +-- MetaWhatsAppProvider   [REQ-002]
              |
              | HTTPS + Meta Authentication
              v
    Official Meta WhatsApp Cloud API
```

The provider is selected through runtime configuration.

REQ-002 introduces real external API calls and may send real WhatsApp messages when the `meta` provider is explicitly selected and valid credentials are supplied.

## Architecture

### Logical Architecture

```text
Secured MCP Boundary from REQ-001
|
+-- send_message
      |
      v
  WhatsAppProvider
      |
      +-- MockWhatsAppProvider
      |
      +-- MetaWhatsAppProvider
              |
              +-- request validation/mapping
              +-- Meta authentication
              +-- timeout and error mapping
              +-- response mapping
              |
              v
          Meta WhatsApp Cloud API
```

The MCP tool depends only on the provider contract. It must not contain Meta-specific HTTP, token, Graph API, template, or response-mapping logic.

### Local Runtime Architecture

```text
Developer Mac
|
+-- Docker Desktop
      |
      +-- Secured Remote MCP Runtime
            |
            +-- HTTPS / Authentication
            +-- Authorization / Rate Limit / Audit
            +-- MCP Server
            +-- WhatsAppProvider selection
            +-- MetaWhatsAppProvider
                    |
                    | outbound HTTPS
                    v
              Meta WhatsApp Cloud API
```

Inbound access remains local unless the developer intentionally supplies another supported local-network configuration. REQ-002 does not require public Internet exposure of the MCP endpoint.

The runtime requires outbound Internet access only when `MetaWhatsAppProvider` is selected.

## Functional Requirements

### Existing MCP Tool Contract

REQ-002 must preserve the REQ-001 MCP-facing tool wherever practical:

```text
send_message(recipient: string, message: string)
```

Inputs:

- `recipient`: required, non-empty string
- `message`: required, non-empty string

The implementation must validate both inputs before provider invocation.

The exact recipient normalization and validation rules for real Meta delivery must be documented and tested. Invalid recipients must fail before an external API call when the invalidity can be detected locally.

### Successful Result

A successful result must remain compatible with the REQ-001 structure:

```json
{
  "status": "success",
  "provider": "meta",
  "message_id": "<identifier-returned-by-meta>",
  "recipient": "+66812345678",
  "message": "Server CPU usage exceeded 90%",
  "timestamp": "2026-10-06T14:30:00Z"
}
```

The `message_id` must be derived from the successful Meta API response rather than generated as a mock value.

The timestamp source and format must be defined consistently in technical design. It must be UTC and machine-readable.

An accepted API request means Meta accepted the message request. It must not be misrepresented as proof that the recipient read the message or that final delivery occurred.

### Supported Message Mode

REQ-002 covers outbound text-oriented messaging through the existing `message` input.

The technical design must select and document the supported Meta delivery mode, including any requirements for:

- a customer-service conversation window
- an approved message template
- test recipients or allowlisted numbers
- locale or template parameters
- Meta account, app, phone-number, and permission configuration

If the chosen Meta mode cannot map cleanly to the existing two-string MCP contract, the design must document the limitation and propose the smallest backward-compatible extension. A breaking change requires explicit approval and is not implied by this requirement.

## Provider Abstraction and Compatibility

`MetaWhatsAppProvider` must implement the `WhatsAppProvider` contract established in REQ-001:

```text
send_message(recipient, message) -> MessageResult
```

The following dependency direction is required:

```text
MCP Tool
   |
   v
WhatsAppProvider contract
   |                 |
   v                 v
Mock provider     Meta provider
```

The MCP layer must not branch on Meta-specific behavior except through provider-neutral results and controlled provider errors.

Provider selection must be configuration-driven. A value equivalent to `mock` selects `MockWhatsAppProvider`; a value equivalent to `meta` selects `MetaWhatsAppProvider`.

Unknown or unsupported provider values must cause a controlled startup/configuration error.

## Official Meta WhatsApp Provider

`MetaWhatsAppProvider` must:

- use the official Meta WhatsApp Cloud API
- use an explicitly configured, currently supported Meta API version
- authenticate using externally supplied Meta credentials
- build a request appropriate to the supported message mode
- enforce a finite request timeout
- map a successful API response into `MessageResult`
- translate expected upstream failures into controlled provider errors
- preserve correlation information across internal logs and audit records where practical
- avoid exposing Meta-specific secrets or sensitive response data

The implementation must not use:

- Twilio
- `whatsapp-web.js`
- Baileys
- WhatsApp Web session automation
- browser automation
- QR-code authentication
- an unofficial reverse-engineered WhatsApp protocol

The Meta API version, endpoint construction, required permissions, and supported payload must be verified against current official Meta documentation during implementation and pinned or configured as defined by the approved design.

## Meta Account and External Prerequisites

The README must clearly identify external prerequisites, which may include:

- a Meta developer account
- a Meta app with WhatsApp capability
- a WhatsApp Business Account or Meta-provided test setup
- a configured phone-number ID
- an access token with the necessary permissions
- an approved or eligible recipient/test number
- an approved message template if the selected messaging scenario requires one

The repository must not contain real account identifiers, access tokens, recipient numbers, template secrets, or exported Meta configuration.

Where Meta offers temporary test assets or tokens, their limitations and expiration behavior must be documented without committing their values.

## Security Inherited from REQ-001

The following REQ-001 controls remain mandatory when the `meta` provider is selected:

- MCP clients connect through HTTPS
- clients authenticate before protected MCP access
- authorization is enforced before provider invocation
- basic local rate limiting is enforced before provider invocation
- each invocation receives a correlation/request ID
- each invocation creates an audit record
- secrets and authorization headers are redacted

Unauthenticated, unauthorized, invalid, or locally rate-limited requests must not reach `MetaWhatsAppProvider` and must not make an external Meta API call.

REQ-002 must add Meta credentials to the existing secret-redaction rules.

The REQ-001 security implementation remains a local-PoC control set and must not be described as production-ready.

## Configuration and Secret Handling

Configuration must include values equivalent to:

- provider selection
- Meta API version
- Meta phone-number ID
- Meta access token or supported credential reference
- request timeout
- supported message mode
- template name and locale where required
- live-test enablement and test recipient where applicable

Environment-specific values must not be hardcoded in application source code.

The repository must provide safe example configuration using placeholders only.

The following must be excluded from Git:

- access tokens
- local environment files containing credentials
- real phone-number IDs or business identifiers when treated as sensitive
- real recipient phone numbers used for testing
- credential exports and debug captures

Missing or inconsistent Meta configuration must cause a controlled startup or provider-selection failure before any delivery attempt.

## Meta API Rate Limits and Delivery Safety

REQ-001 local rate limiting remains active and is distinct from Meta's upstream limits.

The provider must handle upstream throttling as a controlled error and preserve any safe, relevant retry guidance without leaking sensitive response details.

Automatic retries are disabled by default unless the approved design demonstrates how duplicate-message risk is controlled.

If retries are enabled, the implementation must define and test:

- retryable versus non-retryable failures
- maximum attempts
- bounded backoff
- timeout behavior
- duplicate-message risk
- audit events for each attempt and final outcome

The service must not retry authentication, permission, policy, or clearly invalid-recipient failures as if they were transient.

## Audit and Logging

REQ-001 audit and logging requirements continue to apply.

For Meta-backed requests, logs and audit records should make it possible to determine:

- correlation/request ID
- authenticated client identity where available
- authorization outcome
- selected provider
- local validation outcome
- external request outcome
- Meta message ID on success
- controlled provider-error category on failure
- total duration where practical

Logs and audit records must not contain:

- Meta access tokens
- outbound authorization headers
- credential-bearing URLs
- private keys
- raw sensitive Meta error payloads
- message bodies by default
- full recipient values where masking is appropriate

Audit records must distinguish local rejection from an attempted Meta API call.

## Error Handling

Expected Meta provider failures must not terminate the MCP server.

Controlled failure categories must cover at minimum:

- missing or invalid provider configuration
- invalid recipient
- unsupported message mode
- request timeout
- network/connectivity failure
- Meta authentication failure
- Meta authorization or permission failure
- Meta upstream rate limit
- Meta validation or policy rejection
- unsuccessful or malformed Meta response
- unexpected upstream server failure

Client-visible errors must be useful without returning access tokens, raw authorization data, internal stack traces, or unnecessarily detailed upstream payloads.

The provider-error model must preserve enough structured information for tests, audit records, and operations without breaking the provider-neutral MCP contract.

## Docker and Local Deployment

REQ-002 continues to use the local Docker Desktop runtime established by REQ-001.

The normal startup workflow must remain equivalent to:

```sh
docker compose up --build
```

The Docker image must include the dependencies required by `MetaWhatsAppProvider` and must not depend on globally installed host packages.

Meta configuration and credentials must be injected at runtime and must not be baked into the image or committed Compose files.

Switching from `mock` to `meta` must require configuration changes only, followed by a documented restart if required.

The local HTTPS MCP endpoint and REQ-001 security flow must remain available in both provider modes.

## Testing

### Automated Tests

Normal automated tests must not contact Meta or send real messages.

Tests must cover at minimum:

1. `MetaWhatsAppProvider` conforms to the provider contract.
2. Provider selection chooses `mock` or `meta` from configuration.
3. Unknown provider configuration fails safely.
4. Successful Meta request construction for the supported message mode.
5. Successful response mapping, including the Meta message ID.
6. Recipient and message validation before external invocation.
7. Finite timeout behavior.
8. Network failure mapping.
9. Meta authentication and permission failure mapping.
10. Meta rate-limit mapping.
11. Meta validation or policy failure mapping.
12. Malformed or unexpected response handling.
13. Credential, header, recipient, and message redaction.
14. Audit distinction between local rejection and external invocation.
15. Verification that unauthenticated requests do not call Meta.
16. Verification that unauthorized requests do not call Meta.
17. Verification that locally rate-limited requests do not call Meta.
18. Mock-provider regression behavior from REQ-001.
19. Retry and duplicate-risk behavior if retries are enabled.
20. Containerized MCP invocation with a stubbed Meta endpoint.

### Live Integration Test

Any test that can send a real WhatsApp message must be:

- separate from the normal automated test suite
- disabled by default
- explicitly enabled through a documented opt-in
- guarded by an unmistakable real-delivery warning
- configured only through external credentials
- targeted at a developer-controlled test recipient
- designed to minimize cost and unintended messaging
- documented with cleanup and verification steps

A live integration test may be required for final acceptance, but it must never run implicitly during a normal test command, container build, or continuous-integration job.

## Repository and Documentation

The implementation must add appropriate artifacts for:

- `MetaWhatsAppProvider`
- provider-neutral Meta error mapping
- configuration and validation
- stubbed Meta API tests
- optional live integration test
- safe example configuration
- README updates

The README must document:

- relationship to REQ-001
- Meta prerequisites and account setup assumptions
- currently supported API version strategy
- supported text/template delivery mode
- recipient format and eligibility rules
- provider selection
- secret configuration
- local Docker execution
- cost and real-message warnings
- normal offline test execution
- explicit live-test execution
- expected success result
- common controlled failures and troubleshooting
- limitations deferred to REQ-003

## Out of Scope

The following are outside REQ-002 unless separately approved:

- Public production deployment of the MCP endpoint
- Production identity provider integration
- Production secret-management platform
- Production TLS and certificate automation
- Production monitoring and on-call operations
- Infrastructure scaling or high availability
- Incoming WhatsApp webhooks
- Reading inbound messages
- Conversation history or contact synchronization
- Media, document, audio, video, or location messages
- Interactive list, button, flow, or commerce messages
- Template-management user interface
- Bulk campaigns or broadcast messaging
- Marketing automation
- Scheduled messaging
- LLM-generated message content
- Delivery/read-receipt processing
- Twilio or unofficial WhatsApp providers
- Browser or QR-code automation

REQ-002 may document future extension points for these capabilities but must not silently implement them.

## Relationship to Other Requirements

### REQ-001 — Secured Remote MCP Server with Mock WhatsApp Provider

REQ-002 requires REQ-001 and reuses its:

- MCP transport and endpoint
- `send_message` contract
- provider abstraction
- HTTPS and client authentication
- server-side authorization
- local rate limiting
- audit and correlation behavior
- Docker runtime
- configuration and test foundations

REQ-002 must not weaken those controls or create a parallel tool path that bypasses them.

### REQ-003 — Production Remote Deployment and Security Hardening

REQ-003 will move the complete REQ-001 and REQ-002 solution into a remotely hosted, production-oriented environment.

REQ-002 must expose configuration and operational behavior cleanly enough for REQ-003 to provide production secrets, identity, TLS, monitoring, deployment, rollback, and runtime hardening without rewriting `MetaWhatsAppProvider` or the MCP tool contract wherever practical.

## Definition of Done

REQ-002 is complete when a developer can:

1. Start from a completed REQ-001 implementation.
2. Configure official Meta prerequisites and credentials without modifying source code.
3. Select `meta` as the provider through documented configuration.
4. Start the complete runtime locally with Docker Compose.
5. Connect an authenticated and authorized MCP client through HTTPS.
6. Invoke `send_message` using a valid documented Meta test scenario.
7. Observe that REQ-001 authorization, rate limiting, audit, and correlation controls are still enforced.
8. Observe an official Meta API request only after all local checks pass.
9. Receive a provider-neutral success result containing `provider: meta` and the Meta message ID.
10. Confirm real delivery or Meta acceptance using the documented verification procedure.
11. Demonstrate controlled handling of representative Meta failures.
12. Demonstrate that credentials and sensitive values are absent from logs, audit records, client errors, and the repository.
13. Switch back to `mock` through configuration without changing application source code.
14. Run the normal automated test suite without contacting Meta or sending a real message.
15. Run any live test only through its explicit opt-in procedure.

REQ-002 must not be marked complete while any mandatory acceptance criterion fails.

## Open Questions

The following questions must be resolved during technical design or implementation planning:

1. Which currently supported Meta Graph API version will be pinned or configured?
2. Which Meta account and test assets will be used for development and final verification?
3. Will the first real-message scenario use free-form text within an eligible conversation window, an approved template, or another officially supported mode?
4. If a template is required, how will the existing `message` input map to template name, locale, and parameters without breaking the REQ-001 contract?
5. What exact recipient normalization and validation rules will be enforced?
6. Which timestamp represents the provider-neutral result: local acceptance time, Meta response time, or another documented value?
7. Which Meta identifiers may appear in logs or audit records, and which must be masked?
8. What request timeout is appropriate for local development?
9. Will automatic retry remain disabled, or can a safe retry policy be demonstrated for specific failures?
10. How will tests simulate the Meta endpoint: HTTP stubs, a transport adapter, dependency injection, or another deterministic mechanism?
11. What explicit environment flag and safety checks will guard live delivery tests?
12. Which Meta API response fields should be retained internally for troubleshooting without changing the MCP response contract?
13. Are delivery-status webhooks intentionally deferred, or is a separate future requirement needed for them?
14. What real-message cost, account-limit, template-approval, and recipient-eligibility warnings must appear in the README?

All resolved answers must be reflected consistently in the technical design, implementation, automated tests, example configuration, and README.
