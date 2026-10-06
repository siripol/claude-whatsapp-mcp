---
id: REQ-001
title: Remote MCP Server with Mock WhatsApp Provider
priority: high
requires: []
acceptance:
  - The solution exposes a remote MCP server implemented in Python using FastMCP and runs inside a Docker container on the developer's Mac
  - The MCP server uses the current recommended HTTP-based remote transport supported by the selected FastMCP/MCP SDK rather than stdio
  - The MCP server exposes a send_message tool that accepts recipient and message as required inputs
  - Calling send_message invokes the configured WhatsApp provider and returns a structured result containing status, provider, message_id, recipient, message, and timestamp
  - The PoC uses MockWhatsAppProvider only and makes no calls to Meta WhatsApp Cloud API, Twilio, whatsapp-web.js, Baileys, or any other messaging service
  - MockWhatsAppProvider simulates successful delivery without sending a real message or incurring messaging cost
  - Message delivery is abstracted behind a provider interface so MockWhatsAppProvider can later be replaced by MetaWhatsAppProvider without changing the MCP tool contract
  - The MCP server is reachable from the host Mac through a documented local HTTP endpoint after running docker compose up --build
  - Runtime configuration is supplied through environment variables and environment-specific values are not hardcoded in application source code
  - Secrets and local environment files are excluded from Git and are never committed to the repository
  - Empty or whitespace-only recipient and message values are rejected and are not passed to the provider
  - Expected provider failures are returned as controlled errors and do not terminate the MCP server
  - Basic operational logging records server startup, tool invocation, selected provider, delivery result, and message ID without exposing credentials or secrets
  - Automated tests cover MockWhatsAppProvider and core send_message behavior, including success, validation failure, and controlled provider failure
  - The repository contains the Docker artifacts required to build and run the PoC on macOS with Docker Desktop
  - The README documents prerequisites, architecture, local Docker setup, MCP endpoint, tool contract, test execution, and an example invocation
  - A clean clone can be configured from an example environment file and started without modifying application source code
eval_threshold: 70
---

## Context

This PoC establishes the foundation of a remote Model Context Protocol (MCP) server that exposes outbound WhatsApp-style messaging to MCP-compatible AI clients.

The first phase intentionally does not connect to a real WhatsApp service. A mock provider is used so the MCP transport, tool contract, validation, provider abstraction, logging, Docker runtime, error handling, and tests can be implemented without a Meta Business account, WhatsApp phone number, or messaging cost.

The design must preserve a clean extension point for REQ-002, which replaces the mock delivery path with the official Meta WhatsApp Cloud API.

## What

Build a containerized Python Remote MCP Server exposing one outbound messaging tool:

`send_message(recipient: string, message: string)`

The logical flow is:

MCP Client -> Remote MCP Server -> send_message -> WhatsAppProvider -> MockWhatsAppProvider -> Simulated Result

The server runs in Docker on the developer's Mac and is accessed over the current recommended HTTP-based remote MCP transport supported by the selected SDK.

A successful result contains at minimum:

- status
- provider
- message_id
- recipient
- message
- timestamp

No real WhatsApp message is transmitted.

## Functional Requirements

### MCP tool

The server exposes `send_message`.

Inputs:

- `recipient`: required string
- `message`: required string

Both values must reject empty and whitespace-only input.

The MCP layer delegates delivery to a provider abstraction rather than implementing WhatsApp-specific behavior directly.

### Provider abstraction

The design separates the MCP tool from the messaging implementation.

Conceptually:

WhatsAppProvider
- MockWhatsAppProvider — implemented in REQ-001
- MetaWhatsAppProvider — future REQ-002

The provider contract must support behavior equivalent to:

`send_message(recipient, message) -> MessageResult`

### Mock provider

MockWhatsAppProvider must:

- accept recipient and message
- generate a unique mock message ID
- generate a UTC timestamp
- return a successful structured result
- perform no external messaging/network call
- conform to the provider contract intended for the future Meta implementation

## MCP Transport

The primary server transport is remote HTTP, not stdio.

Implementation must use the current recommended HTTP-based remote transport of the selected FastMCP/MCP SDK. The exact endpoint path may follow the SDK convention and must be documented.

During local development it is exposed through Docker to the host Mac, for example:

`http://localhost:8000/mcp`

The exact port is configurable.

## Docker

The complete application must run inside Docker.

The normal development startup command should be:

`docker compose up --build`

The container must listen on an interface that permits Docker port forwarding to the host.

The application must not require globally installed Python dependencies on the Mac after the image is built.

## Configuration

Runtime configuration uses environment variables.

Expected configuration includes values equivalent to:

- MCP host
- MCP port
- provider selection
- log level

The repository provides an example environment configuration while the developer's real local environment file is excluded from Git.

No production Meta credential is required by REQ-001.

## Logging and Error Handling

Operational logging must make server startup, tool invocation, provider selection, delivery result, message ID, and controlled failures observable.

Credentials, access tokens, authorization headers, and future provider secrets must never be logged.

Message-body logging may be enabled for development but the implementation should permit it to be disabled or redacted later.

Expected validation and provider failures must not crash the MCP process.

## Testing

Automated tests must verify at minimum:

- successful mock delivery
- unique message ID generation
- recipient preservation
- message preservation
- timestamp generation
- rejection of empty recipient
- rejection of empty message
- controlled provider failure

Tests must not call an external messaging service.

## Out of Scope

- Real WhatsApp message delivery
- Meta WhatsApp Cloud API integration
- Meta Business verification
- WhatsApp Business Account setup
- Production phone-number registration
- WhatsApp template creation or approval
- WhatsApp pricing or billing
- Twilio
- whatsapp-web.js
- Baileys
- QR-code authentication
- Incoming WhatsApp messages
- Chat/message history
- Media delivery
- Scheduled alerts
- LLM-generated alert content
- RAG or AI-agent workflows
- Public cloud hosting
- Production authentication and authorization
- High availability or horizontal scaling

## Definition of Done

REQ-001 is complete when a developer can clone the repository, configure it from the documented example environment, build and start it with Docker Compose, connect an MCP-compatible client to the local remote MCP endpoint, discover and invoke `send_message`, receive a successful MockWhatsAppProvider response, run the automated tests successfully, and confirm that no external WhatsApp service was contacted.

## Open Questions

- Confirm the exact remote HTTP transport and endpoint convention of the FastMCP/MCP SDK version selected during implementation.
- Decide whether strict E.164 recipient validation belongs in REQ-001 or REQ-002.
- Decide whether message bodies are logged only at debug level.
