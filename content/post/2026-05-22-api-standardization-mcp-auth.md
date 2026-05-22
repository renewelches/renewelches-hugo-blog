---
layout: post
title: "Beyond API Standardization: MCP and Agent Auth"
subtitle: "How a job interview reframed the question for me: from API consistency to MCP capability layers and agent authorization"
date: 2026-05-22
author: "Rene Welches"
description: "A job interview for an API Technical Lead role reframed the question for me: not how to standardize APIs, but how to authenticate and authorize AI agents. Exploring MCP, OAuth 2.1, and the authorization architecture that enterprise AI integrations actually need."
image: "/img/api-standardization-mcp-auth.svg"
URL: "/2026/05/22/api-standardization-mcp-auth/"
tags:
  - MCP
  - OAuth
  - AI Agents
  - API Design
  - Security
categories: [Architecture, AI]
---

I recently went through three rounds of interviews for an API Technical Lead position. The company wanted someone to own their API standardization story end-to-end: define the standards, choose the tooling, drive (enforce) adoption across engineering teams. A classic platform engineering mandate.

By the third interview it became clear that what they were actually trying to solve wasn't API standardization in the traditional sense. They were trying to get their systems _AI-ready_. And that realization opened up a more interesting conversation than the one we'd been having.

_A note on scope: this post is a deep dive into MCP and the human-in-the-loop authorization architecture. It's not a survey of MCP — if you're new to it, the [official docs](https://modelcontextprotocol.io) are a better starting point. The headless agent case gets its own follow-up._

---

## The Framing Problem

The company I was interviewing with had a clear picture of what they wanted to do: take their existing REST APIs, make them more consistent, better documented, maybe add some semantic tagging — and then AI agents would be able to use them.

This is a reasonable instinct but it misses where the ecosystem is heading. The question isn't _how do you make your REST APIs consumable by AI?_ The question is _what is the right interface contract between your systems and an AI agent?_

Those are different questions, and they have different answers.

The first question leads you toward OpenAPI spec improvements, better naming conventions, proper versioning, resource path design, more consistent error envelopes, documentation etc.. Worthwhile work, but not architecturally transformative.

The second question leads you to MCP.

---

## Why MCP Changes the Calculus

The [Model Context Protocol](https://modelcontextprotocol.io) is an open standard — originated by Anthropic, now gaining broad adoption — that defines how AI agents discover and invoke capabilities exposed by external systems. It is, in effect, a protocol designed from first principles for agent-to-system interaction.

What makes MCP architecturally distinct from REST is its unit of composition. REST exposes resources and operations on those resources. MCP exposes _tools_, _resources_, and _prompts_ — primitives designed around how a language model reasons about what it can do and how it should do it.

When an agent connects to an MCP server, it doesn't need to reverse-engineer a REST interface from an OpenAPI spec. It receives a structured manifest of available capabilities, complete with descriptions written for machine comprehension. The protocol handles capability discovery, invocation, and result streaming in a standardized way.

For a company trying to make its systems AI-accessible, the implication is significant: rather than retrofitting their existing API layer, they should be building MCP servers that expose the capabilities agents will actually need. These servers sit in front of existing internal services and translate between the MCP protocol and whatever downstream APIs or data sources they need to call.

This reframes the standardization work entirely. You're not standardizing your REST APIs _for_ AI. You're building a capability layer _above_ your existing APIs that speaks the language agents understand.

---

## The Problem They Hadn't Considered

As the job interview evolved, a harder problem came to my mind — one that I suspect most teams building enterprise MCP integrations haven't fully worked through yet.
How do you authenticate and authorize an agent or a client like Claude Code?

This sounds like a routine question. But - it isn't.

With a human user, authorization flows are well-understood. A person authenticates via an identity provider and receives credentials over OAuth 2.0 — an access token with scopes, and (when OIDC is layered on top) an ID token that conveys identity to the client. Resource servers typically make authorization decisions from the access token's scopes; identity claims from the ID token can feed downstream policy engines like Open Policy Agent when claims-based authorization is in play. Either way, the token directly or indirectly reflects the scope of what the human user is allowed to do. Revocation, session expiry, audit trails — all of these mechanisms exist and are mature.

An AI agent introduces a different set of challenges.

First, what or who triggers an agent. An agent is either invoked on behalf of a human user (human in the loop/chat) or as the consequence of another machine input (event, sensor, cron, triggered by another agent etc.). In case of a user-triggered agent, your orchestration layer routes a task to the agent or a sub-agent. The agent needs to act on behalf of the user and must only have the permissions necessary to complete _its specific task_.
In case of a machine-triggered event the agent (headless agent) needs to act within the boundaries of a certain context — in principle no different from the human-in-the-loop case. However, here we have to handle authentication and authorization differently, because there is no user to delegate from.

Second, agents compose — sub-agents, chained invocations, peer-to-peer orchestration. A team-lead agent might delegate to specialized sub-agents (a loan processing orchestrator coordinating an intake agent, a credit check agent, a risk assessment agent, etc.), and those peers may pass information and state back and forth to complete the workflow. Each hop in that composition is a potential privilege escalation surface if authorization isn't propagated or assigned carefully.

Third, agents are dynamic. The scope of what an agent needs to do may not be fully known at invocation time. An orchestrator might spawn a sub-agent mid-task whose precise capability requirements weren't anticipated when the session began.

We are dealing basically with the two different agent types

- human in the loop and
- headless.

Both types can exist side by side in complex business processes like the loan approval process where the whole intake is mainly agent driven but final review and approval could be with a human in the loop. The auth approaches for these two types diverge significantly — human-in-the-loop flows can leverage user identity directly, while headless agents have no user to delegate from. I'll cover the human-in-the-loop case in this post and the headless case in a follow-up.

---

## MCP Transports: stdio vs HTTP

Before getting into auth, it helps to understand how MCP clients and servers actually connect — because the transport choice determines where the trust boundary sits.

**stdio** is the local transport. The MCP client (e.g., Claude Desktop or Claude Code) spawns the MCP server as a child process and communicates over stdin/stdout. No network is involved. The server lives on the same machine as the client. Authentication is typically not required because process-level trust is sufficient — if you can run the process, you already have local access. This transport is well-suited for developer tooling, filesystem integrations, and local CLI wrappers.
Nevertheless, you can use this pattern to communicate with remote APIs. I created a small Proxmox MCP server which runs locally but connects to the remote Proxmox API. In my case the MCP server has access to an API token which it uses to authenticate against the Proxmox API. You can find the [code here](https://github.com/renewelches/proxmox-stdio-mcp). However, this approach is not very practical in the enterprise context.

**HTTP (Streamable HTTP)** is the remote transport. In this case the client can be an app like Claude Desktop or it can be an agent where the agent is the client itself. The client connects to an MCP server over HTTPS; responses are synchronous or streamed via Server-Sent Events. This is the transport that matters for enterprise integrations, where an MCP server sits in front of internal services and exposes them to agents that may be running in the cloud, on another machine, or inside a different trust domain.

The authentication problems described from here on are primarily HTTP transport problems. For stdio, the client-to-server trust is implicit — process ownership is the boundary. Backend credentials the server needs (API tokens, keys) are typically injected at startup via environment variables or config files, as in the Proxmox example above. This works well for personal tooling but doesn't scale to shared infrastructure. HTTP MCP servers require explicit, verifiable auth for both hops.

---

## Human in the loop - the Authorization Architecture for MCP

### Two authentication hops, not one

When reasoning about auth in MCP over HTTP, there are two distinct hops to consider — and they're easy to conflate.

**Hop 1: MCP client → MCP server.** The MCP specification defines an OAuth 2.1 authorization framework for HTTP-based MCP servers, and explicitly separates the **authorization server** (AS) from the **resource server** (RS). The MCP server is a resource server only — it must not also act as the AS. The client first discovers the MCP server's protected resource metadata (`/.well-known/oauth-protected-resource`, per [RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)), which points to the authorization server. It then performs [dynamic client registration (RFC 7591)](https://datatracker.ietf.org/doc/html/rfc7591), followed by an [authorization code flow with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce) to obtain an access token. That token is then presented as a `Bearer` header on every subsequent request. The MCP server validates the inbound token, checks scopes, and only then dispatches the tool invocation.

```text
[MCP Client] ---(1) Tool request (no token) -----------------------------------> [MCP Server]
[MCP Client] <--(2) 401 Unauthorized + WWW-Authenticate header ----------------- [MCP Server]
[MCP Client] ---(3) GET /.well-known/oauth-protected-resource -----------------> [MCP Server]
[MCP Client] <--(4) Resource metadata (authorization_server URL, scopes) ------- [MCP Server]
[MCP Client] ---(5) GET /.well-known/oauth-authorization-server ---------------> [Auth Server]
[MCP Client] <--(6) Auth server metadata (authorize/token/register endpoints) -- [Auth Server]
[MCP Client] ---(7) POST /register (Dynamic Client Registration, RFC 7591) ----> [Auth Server]
[MCP Client] <--(8) client_id assigned ----------------------------------------- [Auth Server]
[MCP Client] ---(9) Browser redirect to /authorize (+ PKCE code_challenge) ----> [Auth Server]
             <--(10) User authenticates and grants consent
[MCP Client] <--(11) Authorization code (redirect callback) -------------------- [Auth Server]
[MCP Client] ---(12) POST /token (code + PKCE code_verifier) ------------------> [Auth Server]
[MCP Client] <--(13) Access token (Bearer JWT) --------------------------------- [Auth Server]
[MCP Client] ---(14) Retry: tool request + Authorization: Bearer <token> ------> [MCP Server]
[MCP Client] <--(15) Tool response --------------------------------------------- [MCP Server]
```

**Hop 2: MCP server → backend API.** Once the MCP server has validated the client's token, it still needs to call downstream services — your internal CRM API, database, K8s API, or microservices. This is where the design decisions get interesting. The MCP server has a few options:

- **Pass through the user token** directly to backend APIs that accept it. Simple, but only works if the backend shares the same token issuer and trusts the same scopes — and it breaks least privilege: the agent gets the full breadth of the user's permissions, not just what it needs for the task at hand.
- **On-Behalf-Of (OBO) token exchange** — exchange the user's token for a service-specific token downscoped to exactly what the backend call requires. This preserves user identity across the chain while constraining permissions to the minimum necessary.
- **Service account credentials** — the MCP server uses its own identity when calling backends. Simple to implement, but loses the user identity in audit logs and can't enforce per-user permissions downstream.

The OBO flow exists precisely to solve Hop 2 while preserving user identity all the way to the backend. The right answer here draws on **OAuth 2.0 Token Exchange** as defined in [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693) — the standardized mechanism that On-Behalf-Of (Microsoft Entra's branding) is one implementation of. RFC 8693 distinguishes two modes: **impersonation** (the new token preserves the original subject with no trace of the actor) and **delegation** (the original subject is preserved AND an `act` claim identifies the agent in the chain). For agent contexts, delegation is the right default — it keeps the agent visible in audit logs, which matters when something goes wrong and you need to know which agent actually made the call.

The core idea: when a caller (an agent, in this case) needs to act on behalf of a subject (the user), it can exchange a token it already holds for a new, downscoped token that:

1. Retains the identity of the original subject (so audit logs correctly attribute actions to the human), with an `act` claim identifying the agent that performed the action
2. Is restricted to the scopes necessary for the specific task
3. Has a shorter expiry appropriate to the operation

This pattern assumes you have an access token to exchange. Some deployments don't — `kubectl` is the canonical example: it sends an OIDC **ID token** directly to the API server as a bearer credential, and the API server maps ID token claims to RBAC via `--oidc-username-claim` / `--oidc-groups-claim`. ID tokens were never designed to be presented to resource servers (they're for clients to verify _who the user is_), and there's no exchange-capable authorization server in that path to downscope them. RFC 8693 itself supports ID tokens as input and output token types — so the limitation isn't the spec, it's the deployment shape. I might do a third post about this kind of problem.

In an agent system, token exchange can happen at two distinct points — and a mature setup will often use **both**. They are complementary, not redundant: one controls what the agent can see at the MCP layer, the other controls what the MCP server can do at the backend layer.

**Exchange Point 1 — Agent-side, pre-MCP.** Before a sub-agent ever talks to an MCP server, the orchestrator exchanges the user's broad token for a narrowly-scoped, short-lived token bound to the sub-agent's specific task. The MCP server then receives a token that already reflects least privilege for this task.

```text
User authenticates → receives user token (broad scopes)
      ↓
Orchestrator Agent presents user token to Authorization Server
Requests downscoped token for sub-agent task X
      ↓
Authorization Server validates, issues task-scoped token
(subject = user, scopes = {task_x_scopes_only}, exp = short TTL)
      ↓
Sub-agent presents task-scoped token to MCP Server
MCP Server validates, executes tool invocation
```

**Exchange Point 2 — MCP server-side, Hop 2.** Once the MCP server has validated the inbound token, it still needs to call backends. Rather than passing the inbound token through, the MCP server itself performs an RFC 8693 token exchange — taking the inbound token and exchanging it for a backend-specific token, downscoped to exactly what _this_ tool invocation needs and audience-bound to the specific backend.

```text
[Sub-Agent] ---(1) Tool call + inbound token (task-scoped) -------------------> [MCP Server]
[MCP Server] -- validates inbound token (sub, scope, aud=MCP) --
[MCP Server] ---(2) POST /token  (RFC 8693 token exchange) -------------------> [Auth Server]
              subject_token = inbound token
              resource      = https://backend.example/    (RFC 8707)
              scope         = "loan:applicant:read"
              actor_token   = MCP server's own credential  (delegation → `act` claim)
[MCP Server] <--(3) Backend-scoped token (sub=user, act=MCP, aud=backend) ---- [Auth Server]
[MCP Server] ---(4) Backend call + Authorization: Bearer <backend token> -----> [Backend API]
[MCP Server] <--(5) Backend response ----------------------------------------- [Backend API]
[MCP Server] ---(6) Tool result ----------------------------------------------> [Sub-Agent]
```

Without Exchange Point 1, every sub-agent reaches the MCP server with the full breadth of the user's permissions — least privilege fails at the agent layer. Without Exchange Point 2, the MCP server's privilege at the backend is whatever the inbound token happens to confer — least privilege fails at the service layer. You need both to actually achieve end-to-end least privilege across an agent chain.

The MCP server itself also needs to enforce scope validation per tool. Each tool in the MCP server's manifest should have a declared scope requirement, and the server should reject invocations where the presented token doesn't satisfy those requirements — even if the token is otherwise valid.

This is analogous to how OAuth-protected APIs work today, but applied at the tool invocation level rather than the endpoint level.

### Scope Design for MCP Tools

Scope design is where a lot of the real work lives. A few principles:

**Prefer fine-grained scopes over coarse-grained ones.** `loan:applicant:read` is better than `loan:read`. The former can be granted to the intake agent that only needs to retrieve applicant data; the latter opens the entire loan read surface — including credit scores, risk assessments, and approval decisions the intake agent has no business seeing.

**Model scopes around business capabilities, not technical endpoints.** Agents reason about tasks, not HTTP verbs. `loan:credit:check` is a more natural scope for the credit check agent than `PUT /loans/{id}/credit-score`.

**Consider temporal scoping.** Some operations should only be permitted within a specific execution context. Token exchange allows you to issue tokens with a TTL calibrated to the expected duration of a task — a 5-minute TTL for a quick lookup is different from a longer TTL for a multi-step workflow.

**Treat sub-agent tokens as a separate identity tier.** Your authorization server should be able to distinguish between tokens issued to humans, to first-party orchestrators, and to sub-agents. A first-party orchestrator is an agent your organization built and operates with a known, stable identity — the loan processing orchestrator is a good example: it's a long-lived, trusted principal that your team deploys and controls. Sub-agents it spawns (the intake agent, the credit check agent) are more ephemeral and should carry narrower, shorter-lived tokens. This three-tier distinction matters for audit, for revocation policy, and for anomaly detection.

### The Audience Binding Problem

One subtlety worth calling out: token audience binding.

When an authorization server issues a token via the OBO flow, it should bind the token to the intended audience — the specific MCP server that will consume it. This prevents a token issued for MCP Server A from being replayed against MCP Server B.

In OAuth 2.0 terms, this is the `aud` claim in the token. The standardized way to _request_ a correctly-bound token is [RFC 8707 (Resource Indicators)](https://datatracker.ietf.org/doc/html/rfc8707): the client tells the AS which resource server the token is for via a `resource` parameter, and the AS scopes the `aud` claim accordingly. The [MCP authorization spec (2025-06-18)](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization#resource-parameter-implementation) makes this a hard requirement: clients **MUST** include the `resource` parameter in authorization and token requests, and MCP servers **MUST** validate that presented tokens were specifically issued for them. Without this binding, a compromised agent that obtains a valid token for one MCP server could attempt to use it against others.

### Refresh Tokens for Long-Running Agents

Long-running agents face a question humans don't: access tokens expire. If an agent's task outlives its token's TTL, you have to decide whether the agent holds a refresh token and silently renews, whether it re-prompts the user, or whether sessions are bounded by the token's lifetime. Each choice has trade-offs — refresh tokens broaden the agent's blast radius if leaked; re-prompting breaks autonomous workflows; hard session limits force task chunking. There's no universal right answer, but the decision should be explicit rather than inherited from whatever the OAuth library defaults to.

### Consent UX for First-Party vs Third-Party MCP Servers

The dynamic client registration + authorization code flow technically includes a user consent step. For first-party MCP servers — ones your organization built and operates — you almost certainly want to pre-approve them in the authorization server, so users aren't pestered with a consent dialog every time a new sub-agent spins up. For third-party MCP servers (an external SaaS exposing an MCP surface, say), you absolutely want the consent dialog, and probably scope-by-scope rather than blanket approval. Your authorization server should be able to distinguish trust tiers and apply consent policy accordingly.

---

## What This Means for API Standardization Strategy

Coming back to the interview context: the company's original goal was to standardize their APIs. That's still a valid goal — having consistent API design, versioning, and documentation practices matters for developer experience and maintainability.

But the AI-readiness goal is distinct, and pursuing it through API standardization alone leaves the hardest problems unaddressed.

A more coherent strategy looks like this:

> **A note on prerequisites.** In the interview context, the real blocker behind their AI-readiness goal turned out to be auth fragmentation — different IdPs, ad-hoc API keys, service-specific auth schemes accumulated over the years across teams. No amount of API standardization or MCP tooling closes that gap. That's why Layer 0 below is prerequisite work, not an afterthought.

**Layer 0 — Unified identity foundation.** Establish a single token issuer and a shared authorization server that all services can validate tokens against. Consolidate fragmented IdPs, retire ad-hoc API keys, and agree on a common auth dialect across teams. Without this, token exchange flows have nothing to anchor to and scope propagation breaks at every service boundary.

**Layer 1 — Internal API hygiene.** Standardize internal REST/gRPC APIs for consistency, discoverability, and developer experience. OpenAPI-first design, consistent error envelopes, versioning policy. This work has independent value regardless of AI.

**Layer 2 — MCP capability surface.** Build MCP servers that expose business capabilities to AI agents. These servers are first-class software artifacts with their own lifecycle, not just thin wrappers. They define what agents can do, using the vocabulary agents understand.

**Layer 3 — Agent authorization infrastructure.** Implement token exchange flows, scope registries, and per-tool authorization enforcement. This is the layer most teams haven't built yet, and it's the one that determines whether your AI integration is secure at any meaningful scale.

The third layer is the one that takes the most time to get right. It requires coordination between platform, security, and identity teams. It requires decisions about scope design that will be hard to reverse. And it requires buy-in on a mental model — treating agents as a distinct principal type, not just another API client — that many organizations haven't internalized yet.

---

## Closing Thought

The most interesting part of that final interview wasn't what they asked me. It was the moment when I realized why they wanted API standardization (AI-readiness) and it hit me that their actual biggest problem would be authenticating and authorizing AI agents to access their APIs.

I think API standardization is important and a great goal. But the focus and priorities in this case were, in my view, misplaced. Instead of focusing on API standardization the company should focus on standardizing auth and work on the interface contract between their systems and AI agents as a first-class architectural concern (MCP) — and build the infrastructure to support it securely, from the ground up.

That's a completely different problem than the one they posted the job for.
