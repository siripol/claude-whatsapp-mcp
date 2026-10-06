---
id: REQ-002
title: Official Meta WhatsApp Cloud API Provider
priority: high
requires:
  - REQ-001
acceptance:
  - A MetaWhatsAppProvider implements the provider contract established by REQ-001 without requiring a breaking change to the MCP send_message tool contract
  - The provider sends outbound WhatsApp messages only through the official Meta WhatsApp Business Platform / Cloud API
  - Meta credentials and identifiers are supplied through environment or secret configuration and are never hardcoded in source code
  - Access tokens, authorization headers, and other secrets never appear in application logs, test output, committed files, or delivered artifacts
  - The implementation supports the message form required for approved business-initiated WhatsApp delivery, including template-based delivery where required by Meta policy
  - Provider responses are mapped into the common structured MessageResult used by REQ-001
  - Meta API errors, authentication failures, invalid recipients, rejected templates, rate limits, and transient network failures are handled as controlled provider errors and do not crash the MCP server
  - The provider preserves the Meta message identifier when Meta accepts a message for delivery
  - Recipient restrictions can be configured so development and testing cannot accidentally send messages to arbitrary phone numbers
  - MockWhatsAppProvider remains available for local development and automated tests without requiring Meta credentials
  - Automated tests cover Meta provider request construction and response/error mapping without sending paid or uncontrolled real messages
  - An explicitly initiated integration test can validate delivery through a configured Meta test or approved development environment
  - README documentation explains Meta prerequisites, required configuration, provider selection, template requirements, safe test procedure, and known external dependencies
eval_threshold: 70
---

## Context

REQ-001 establishes a remote MCP server and a provider abstraction using MockWhatsAppProvider. REQ-002 adds real outbound WhatsApp delivery through the official Meta WhatsApp Business Platform / Cloud API while preserving the MCP-facing contract.

This requirement intentionally treats Meta as an external provider. Meta-specific API details, credentials, message templates, policy constraints, and error behavior must remain isolated from the MCP tool layer wherever practical.

The goal is to prove that an MCP client can invoke the same `send_message` capability developed in REQ-001 while the underlying provider performs a real, authorized WhatsApp delivery.

## What

Implement `MetaWhatsAppProvider` behind the WhatsAppProvider abstraction created by REQ-001.

Target flow:

MCP Client -> Remote MCP Server -> send_message -> WhatsAppProvider -> MetaWhatsAppProvider -> Meta WhatsApp Cloud API -> WhatsApp Recipient

The existing MockWhatsAppProvider remains supported.

Provider selection is configuration-driven; application source code must not need modification when switching between mock and Meta environments.

## Meta Provider

MetaWhatsAppProvider is responsible for:

- constructing valid requests for the official Meta WhatsApp API
- applying authentication
- using the configured WhatsApp phone-number identifier and related Meta configuration
- sending supported outbound messages
- mapping Meta responses into the common MessageResult
- mapping provider failures into controlled application errors

Meta-specific HTTP payloads and response structures must not leak unnecessarily into the MCP tool contract.

## Message Delivery

The implementation must account for WhatsApp's distinction between messages that can be sent within an active customer-service conversation window and business-initiated messages that require approved templates.

The PoC's primary use case is proactive outbound alerts. Therefore the implementation must support the approved template-based delivery mechanism required for that use case.

If the provider supports more than one delivery mode, the behavior must be explicit and deterministic rather than silently selecting an unsupported message form.

## Configuration and Secrets

Meta configuration is externalized.

Values may include, depending on the current official API contract:

- access token
- WhatsApp phone-number ID
- WhatsApp Business Account-related identifiers when required
- API version
- approved template name
- template language
- recipient allowlist for development

Actual secret values must never be committed.

The repository may contain names/placeholders in `.env.example`, but never usable credentials.

## Recipient Safety

Real-message environments must support restricting recipients to an explicitly configured development/test allowlist.

A request targeting a recipient outside that allowlist must be rejected before calling Meta when the restriction is enabled.

This safeguard exists to prevent accidental outbound messaging during development and testing.

## Error Handling

The provider must distinguish and expose useful controlled failures for conditions including:

- invalid or expired authentication
- invalid configuration
- invalid recipient
- template rejection or template mismatch
- provider rate limiting
- Meta API validation errors
- transient HTTP/network failures

Secrets and full authorization data must not be included in error messages.

Retry behavior, if introduced, must be bounded and must not create uncontrolled duplicate messages.

## Testing

Normal automated tests must not depend on paid real-message delivery.

Tests should use mocks/fakes at the Meta HTTP boundary to validate:

- request construction
- authentication-header construction without exposing the secret
- successful response mapping
- Meta message ID preservation
- API error mapping
- timeout/network error handling
- recipient allowlist enforcement

A real integration test must be opt-in and require explicit environment configuration.

It must never run automatically as part of the normal unit-test suite.

## Out of Scope

- Public production deployment of the MCP server
- Internet-facing MCP authentication
- Production secret-management platform
- High availability
- Horizontal scaling
- Production monitoring/alerting
- Incoming WhatsApp webhook processing
- Two-way chatbot conversations
- Reading chats or message history
- Media messaging unless separately approved
- Bulk marketing campaigns
- Contact-list management
- AI-generated content or RAG
- Unofficial WhatsApp libraries or browser automation

These concerns are either covered by REQ-003 or require separate future requirements.

## Definition of Done

REQ-002 is complete when the REQ-001 MCP server can be configured to use MetaWhatsAppProvider, the existing `send_message` capability can deliver an authorized test message through the official Meta WhatsApp Cloud API to an approved recipient, the returned provider message ID is preserved, failure scenarios are controlled, automated tests pass without uncontrolled real delivery, and MockWhatsAppProvider remains usable without Meta configuration.

## Open Questions

- Confirm the current Meta API version and supported message payloads at implementation time.
- Confirm which approved template will be used for the outbound-alert integration test.
- Decide whether the MCP tool remains exactly `recipient + message` or gains an optional non-breaking template/context input for Meta template parameters.
- Define the development recipient-allowlist configuration format.
- Define bounded retry behavior for transient Meta failures.
