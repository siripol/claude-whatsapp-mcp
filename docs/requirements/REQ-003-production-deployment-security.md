---
id: REQ-003
title: Production Remote MCP Deployment and Security
priority: high
requires:
  - REQ-001
  - REQ-002
acceptance:
  - The Dockerized MCP server is deployable to a remote environment and is reachable through a stable HTTPS endpoint rather than a developer-local localhost endpoint
  - All externally accessible MCP traffic is protected by TLS and plaintext public MCP access is not used
  - The remote MCP endpoint requires authentication and unauthenticated requests cannot discover or invoke messaging tools
  - Authorization controls restrict access to the outbound messaging capability according to configured policy
  - Real WhatsApp delivery is restricted to an explicit recipient policy or allowlist appropriate to the deployment
  - Production secrets are supplied through an appropriate secret-management or protected runtime configuration mechanism and are not stored in the container image, source repository, Compose files, or application logs
  - Logs provide sufficient information to audit MCP requests and message-delivery outcomes without exposing access tokens, authorization headers, provider secrets, or unnecessary sensitive message content
  - Production logging includes a correlation/request identifier sufficient to trace an MCP invocation through the provider result
  - Rate limiting or equivalent abuse protection prevents an authenticated client from generating uncontrolled message volume
  - Health/readiness behavior allows the runtime platform to determine whether the MCP service is available without exposing secrets or provider credentials
  - Application and container processes run with least practical privilege and do not require privileged Docker execution
  - Deployment configuration supports controlled restart and recovery after process or container failure
  - Production provider failures remain controlled and do not expose internal stack traces or secrets to remote clients
  - A deployment runbook documents deployment, configuration, secret rotation, rollback, health verification, log inspection, and incident shutdown procedures
  - Security-focused tests validate authentication, authorization, recipient restrictions, secret redaction, and rate-limit behavior
  - REQ-001 mock mode remains available for local development without weakening production security controls
eval_threshold: 70
---

## Context

REQ-001 creates the remote MCP application and Docker runtime. REQ-002 adds real WhatsApp delivery through MetaWhatsAppProvider.

REQ-003 moves that application from a developer-local Docker environment to a remotely reachable, production-oriented MCP service.

An internet-accessible MCP endpoint capable of sending real WhatsApp messages creates a materially different risk profile from a localhost PoC. Authentication, authorization, secret handling, recipient controls, abuse protection, auditability, TLS, and operational recovery are therefore part of production readiness rather than optional follow-up work.

This requirement does not mandate a specific cloud vendor. The deployment design should remain portable where practical.

## What

Deploy the Dockerized MCP server to a remote runtime and protect it for controlled real-world use.

Target flow:

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
    v
WhatsAppProvider
    |
    v
MetaWhatsAppProvider
    |
    v
Meta WhatsApp Cloud API

The deployment must expose only the interfaces necessary for MCP operation and health management.

## HTTPS

The public MCP endpoint must use HTTPS.

TLS termination may be provided by the application platform, managed ingress, reverse proxy, load balancer, or another appropriate infrastructure component.

Production documentation must clearly identify where TLS terminates and how certificates are managed.

## Authentication

Remote MCP access must require authentication.

The exact mechanism is an implementation decision selected according to the MCP SDK/client capabilities and deployment environment current at implementation time.

Authentication material must be externalized and treated as secret.

Unauthenticated callers must not be able to invoke `send_message`.

## Authorization

Authentication alone is insufficient.

The system must apply authorization policy before allowing a client to invoke real outbound messaging.

The initial deployment may use a simple client/role policy if appropriate, but the authorization boundary must be explicit and testable.

## Recipient Controls

Production real-message delivery must enforce an explicit recipient policy.

For an initial controlled deployment this may be an allowlist.

Recipient restrictions must be enforced server-side and must not depend solely on client behavior or prompts supplied to an LLM.

Rejected recipients must not reach MetaWhatsAppProvider.

## Secret Management

Production secrets must not be:

- committed to Git
- copied into the Docker image
- embedded in application source
- stored in documentation
- printed in logs
- returned in MCP errors

The deployment should use the runtime platform's protected secret/configuration capability or an appropriate secret-management service.

Secret rotation must be documented.

## Logging and Auditability

Production logs must make it possible to determine:

- when an MCP request occurred
- which authenticated client/principal initiated it, where supported
- which tool was invoked
- whether recipient policy allowed the request
- which provider handled the request
- provider message ID where available
- final success/failure state
- correlation/request ID

Logs must avoid unnecessary message-body content by default.

Secrets and authorization headers must always be redacted.

## Abuse Protection

The remote service must include rate limiting or equivalent controls appropriate to the expected workload.

The objective is to prevent accidental loops, compromised clients, or malformed automation from generating uncontrolled WhatsApp messages.

Limits must be configurable rather than hardcoded where practical.

## Health and Runtime Operations

The deployed service must expose a safe mechanism for runtime health/readiness checking.

Health responses must not expose credentials, secret configuration, recipient lists, or unnecessary internal details.

The deployment must define restart/recovery behavior for process or container failure.

## Container Security

The production container must follow least-practical-privilege principles.

It must not require privileged Docker mode.

Where practical, the application should run as a non-root user and the image should contain only dependencies required at runtime.

## Deployment Portability

No specific cloud provider is mandated by this requirement.

A deployment may use a managed container service, VM/container host, Kubernetes, or another suitable runtime provided all mandatory acceptance criteria are met.

Provider-specific infrastructure should be isolated from core application code where practical.

## Testing

Security and deployment validation must cover at minimum:

- HTTPS availability
- rejection of unauthenticated MCP access
- rejection of unauthorized tool invocation
- recipient-policy enforcement
- secret/log redaction
- rate-limit behavior
- controlled provider failure
- health/readiness behavior
- restart/recovery behavior

Tests must avoid uncontrolled real WhatsApp delivery.

## Operations Runbook

Production documentation must cover:

- deployment procedure
- required runtime configuration
- secret provisioning
- secret rotation
- health verification
- log inspection
- restart procedure
- rollback procedure
- emergency shutdown / disabling outbound delivery
- switching to mock mode in a non-production environment

## Out of Scope

Unless separately approved, REQ-003 does not include:

- multi-region active-active deployment
- guaranteed zero-downtime deployment
- Kubernetes as a mandatory platform
- autoscaling to high-volume marketing workloads
- incoming WhatsApp webhook processing
- two-way chatbot conversations
- bulk messaging campaigns
- RAG
- AI-agent orchestration
- business analytics dashboards
- enterprise SIEM integration
- formal regulatory certification

## Definition of Done

REQ-003 is complete when an authorized MCP client can connect to the remote HTTPS endpoint, authenticate, invoke the permitted outbound messaging tool, pass server-side recipient controls, deliver through MetaWhatsAppProvider, and obtain a controlled result; unauthorized or disallowed requests are blocked; production secrets remain protected; audit logs and rate controls are operational; security tests pass; and the deployment/runbook supports health verification, restart, rollback, and emergency shutdown.

## Open Questions

- Select the target remote hosting platform.
- Select the MCP-compatible authentication mechanism based on the current MCP specification and client capabilities.
- Define the initial authorization model.
- Define production recipient policy and ownership of allowlist changes.
- Define initial rate-limit values.
- Decide whether a reverse proxy/managed ingress is required or whether the hosting platform terminates TLS directly.
- Define log retention and production observability requirements.
