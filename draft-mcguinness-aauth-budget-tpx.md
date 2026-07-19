---
title: "TPX-A: An AAuth Profile for Metered LLM Inference Grants"
abbrev: "TPX-A"
category: std

docname: draft-mcguinness-aauth-budget-tpx-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - aauth
 - agent
 - budget
 - llm
 - inference
 - metering
venue:
  github: "mcguinness/draft-mcguinness-aauth-budget"
  latest: "https://mcguinness.github.io/draft-mcguinness-aauth-budget/draft-mcguinness-aauth-budget-tpx.html"

author:
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com

normative:
  I-D.draft-hardt-oauth-aauth-protocol:
    title: "AAuth Protocol"
    author:
      -
        ins: D. Hardt
        name: Dick Hardt
    date: 2026
    seriesinfo:
      Internet-Draft: draft-hardt-oauth-aauth-protocol-08
  I-D.draft-mcguinness-aauth-budget:
    title: "Budgeted Missions for AAuth"
    target: https://mcguinness.github.io/draft-mcguinness-aauth-budget/draft-mcguinness-aauth-budget.html
    author:
      -
        ins: K. McGuinness
        name: Karl McGuinness
    date: 2026

informative:
  RFC9421:
  TPX:
    title: "TPX v0.2: An OAuth 2.0 profile for metered LLM inference grants"
    target: https://tokenpony.dev/spec/
    date: 2026

--- abstract

TPX enables LLM applications to operate without embedded provider
credentials: a person grants an application a metered token budget
from a provider of their choice, and the person handles payment
directly. TPX v0.2 is an OAuth 2.0 profile of that pattern. This
document, TPX-A, is its AAuth-native sibling: the application is an
agent with its own cryptographic identity, consent happens at the
person's Person Server, and the grant is an approved AAuth mission
carrying a budget defined by Budgeted Missions for AAuth. TPX-A pins
what that extension leaves to profiles: the credit as the published
pricing unit, a model restriction, the OpenAI-compatible inference
API surface with its usage-accounting member, the balance model, and
identity minimization.

--- middle

# Introduction

TPX enables LLM applications to operate without embedded provider
credentials. A person grants an application a metered token budget
from a provider of their choice, and the person handles payment
directly with the provider. TPX v0.2 {{TPX}} profiles OAuth 2.0 for
that pattern.

This document, TPX-A, is the AAuth-native sibling. The application
is an AAuth agent with its own cryptographic identity
{{I-D.draft-hardt-oauth-aauth-protocol}}, consent happens at the
person's Person Server (PS), and the grant is an approved AAuth
mission carrying a cap defined by Budgeted Missions for AAuth
{{I-D.draft-mcguinness-aauth-budget}} ("AAuth-Budget"), which
supplies the cap mechanics: the `budgets` mission member, the
`budget` auth-token claim, the budget-state endpoint, and the
`budget_exhausted` signal.

TPX-A pins the five things AAuth-Budget leaves to profiles:

- the credit as the published pricing unit ({{credit}});
- the `models` member on a budgets entry ({{models}});
- the inference API surface, including where the budget endpoint
  lives and the `credits_charged` usage member ({{api}});
- the balance model behind the cap, with the `balance_exhausted`
  signal ({{balances}}); and
- identity minimization in auth tokens ({{identity}}).

Everything else is AAuth and AAuth-Budget. TPX-A uses AAuth's
federated access mode: the provider operates both the resource and
its Access Server (AS), while the person uses a PS of their choice.
There is no client registration (the agent identifier is verified by
construction), no redirect, no authorization code, no PKCE, no
refresh token, and no optional sender-constraining: every token is
key-bound and every request is signed, per AAuth. {{mapping}} maps
each TPX v0.2 mechanism to its TPX-A equivalent or records its
retirement.

## Applicability

This document tracks draft-hardt-oauth-aauth-protocol-08, an
individual Internet-Draft; a change to AAuth's surfaces revises this
document. It targets deployments where the person operates or
chooses a Person Server and the provider serves an OpenAI-compatible
inference API as an AAuth resource backed by the provider's Access
Server.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses Person Server (PS), Access Server, agent, agent
token, resource token, auth token, mission blob, `s256`, and the
`AAuth-Mission` header as defined by
{{I-D.draft-hardt-oauth-aauth-protocol}}, and budget, payee, the
`budgets` member, the `budget` claim, `budget_endpoint`, and
`budget_exhausted` as defined by
{{I-D.draft-mcguinness-aauth-budget}}. It additionally uses:

Person:
: The human who maintains a provider balance, chooses a Person
  Server, and approves missions. TPX v0.2's User.

Agent:
: The LLM application, an AAuth agent with its own identity. It
  holds its own signing key but never holds a provider credential or
  shared client secret. TPX v0.2's App.

Provider:
: A single operator serving an AAuth resource (an OpenAI-compatible
  inference API), its Access Server, and person balances. The
  resource and Access Server can use different origins. TPX v0.2's
  Provider.

Grant:
: A person's approval of one agent's access for one budget,
  implemented as an approved AAuth mission and the auth tokens
  issued under it. The budget lives on the mission, not on any
  token.

Credit:
: The provider's published pricing unit: 1 credit = US$0.000001. A
  budget of 100000 credits caps spend at $0.10.

All JSON shown in this document is non-normative and illustrative;
the member definitions in the surrounding text are authoritative.

# Design Goals

1. Agents work with any person-named provider and any conformant
   Person Server; the agent hard-codes only AAuth.
2. Auth tokens reveal no person identity or stable cross-grant
   correlation handle to agents.
3. Budgets enforce hard damage caps; a compromised agent cannot
   exceed the remaining budget.
4. Substrate reuse: TPX-A rides AAuth's endpoints, tokens,
   signatures, and security analysis, and AAuth-Budget's cap
   mechanics, and adds no new authorization flow.

# Protocol Flow {#flow}

1. The person names a provider to the agent.
2. The agent proposes a mission at the person's PS (the `ps` claim
   of its agent token names it): a description, tools, and a
   `budgets` entry naming the provider, an amount, and any model
   restriction.
3. The PS clarifies as needed, authenticates the person, and
   renders consent per AAuth-Budget: the agent's verified identity,
   the amount with its currency, and the model restriction. The
   person approves, possibly with a lower amount. Approval is
   naturally asynchronous per AAuth; nothing blocks on a browser
   redirect, because there is none.
4. The PS returns the approved mission blob and its reference
   (`approver`, `s256`).
5. The agent requests inference; the provider answers 401 with a
   resource token, copying the mission reference from the
   `AAuth-Mission` header.
6. The agent presents the resource token at the PS token endpoint
   under the mission.
7. The PS verifies the mission is active and the resource matches
   the granted entry, then federates to the provider's Access Server,
   conveying the granted entry in AAuth-Budget's `budget` federation
   parameter. The Access Server applies provider policy and returns
   an auth token carrying the mission reference and granted `budget`
   claim, but no person identifier.
8. The agent makes signed inference requests. The provider prices
   actual usage, debits the grant keyed by the mission reference,
   and reports `credits_charged`.
9. When a request's maximum charge cannot fit the remainder, the
   provider fails closed with 402 before inference. The agent can use
   a smaller finite output bound or propose completion with total
   spend; increasing the cap is a new mission, per AAuth-Budget.

# The Credit {#credit}

Budgets are committed in currency per AAuth-Budget; TPX-A providers
price in credits at a fixed, exact conversion: 1 credit =
US$0.000001. A TPX-A `budgets` entry MUST use `USD` and MUST have no
more than six fractional digits in `amount`, so every conforming
amount converts to a whole number of credits. The conversion is the
exact decimal amount multiplied by 1000000. An agent proposing a 50000-credit budget
therefore proposes `{"amount": "0.05", "currency": "USD"}`. The
person consents in money; the API surface reports in credits; the
conversion is exact in both directions.

The converted cap MUST NOT exceed 9007199254740991 credits (2^53-1).
Every credit-count JSON integer defined by this profile is in the
inclusive range 0 through that value, so common JSON implementations
can preserve it exactly.

Model discovery is API surface, not AAuth metadata:
`GET {resource}/v1/models` lists available models with per-token
rates in credits for fresh input, cached input, and output.

# The models Member {#models}

TPX-A adds one OPTIONAL member to a `budgets` entry:

`models`:
: A non-empty array of unique, non-empty model identifier strings the
  grant is limited to. Identifiers are compared by exact string
  match. An absent member means all models.

AAuth-Budget's fail-closed rule applies: a PS that does not
recognize `models` rejects the proposal, and a provider that cannot
enforce it rejects the token. The PS renders the restriction at
consent. If a proposal contains `models`, the granted value MUST be a
non-empty subset of the proposed array; removing the member would
broaden the grant and is forbidden. If the proposal omits `models`,
the PS MAY add it as an attenuation.

# Inference API {#api}

"OpenAI-compatible" in this document identifies the familiar chat
completions request, response, and SSE shapes; it does not incorporate
an evolving external API by reference. TPX-A interoperability covers
the paths and the authentication, model-rate, usage, metering, state,
and error requirements defined below. Providers MUST document any
other supported request or response fields, and agents MUST NOT assume
that unspecified optional features are present.

## Authentication {#api-auth}

Every inference request is signed with the agent's key and carries
the auth token, whose `mission` and `budget` claims bind the request
to the grant:

~~~ http-message
POST /v1/chat/completions HTTP/1.1
Host: api.tokenpony.dev
Content-Type: application/json
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key");created=1784387100
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="<auth token>"
~~~

The provider verifies the HTTP Message Signature {{RFC9421}} against
the auth token's `cnf.jwk` and verifies the token per AAuth. If an
`AAuth-Mission` header is present, it MUST equal the token's
`mission` claim. A model outside the granted restriction is refused
with `model_not_allowed` ({{api-errors}}).

All inference access is budgeted. The provider MUST refuse an
inference request whose auth token lacks `inference` among its
space-separated `scope` values, a `mission` claim, or a conformant
`budget` claim, per AAuth-Budget's fail-closed rule.

## Required Endpoints {#api-endpoints}

Relative to the resource identifier:

- `GET /v1/models`: available models with per-token rates in credits,
  as defined below. It MAY be served without authentication.
- `GET /grant`: the AAuth-Budget budget-state endpoint
  ({{api-grant}}). The provider publishes it as `budget_endpoint`
  in its resource metadata.
- `POST /v1/chat/completions`: OpenAI-compatible, streaming (SSE) and
  non-streaming.

`POST /v1/chat/completions` MUST accept an OPTIONAL
`max_completion_tokens` positive JSON integer and MUST treat it as a
hard upper bound on generated output tokens. When it is absent, the
provider MUST apply and document a finite default.

For example, the resource metadata contains:

~~~ json
{
  "issuer": "https://api.tokenpony.dev",
  "jwks_uri": "https://api.tokenpony.dev/.well-known/jwks.json",
  "access_mode": "auth-token",
  "budget_endpoint": "https://api.tokenpony.dev/grant"
}
~~~

`GET /v1/models` returns an OpenAI-compatible list. Every model entry
MUST include `id` and a `credits_per_token` object:

~~~ json
{
  "object": "list",
  "data": [
    {
      "id": "pony-8b",
      "object": "model",
      "credits_per_token": {
        "input": "1",
        "cached_input": "0.25",
        "output": "5"
      }
    }
  ]
}
~~~

`input`, `cached_input`, and `output` are REQUIRED non-negative
values in AAuth-Budget's decimal string syntax. `cached_input`
MUST NOT exceed `input`. The rates in effect when a request is
admitted apply for the whole request; a provider MUST NOT change a
rate during an admitted request. Rate changes can apply to later
requests.

## Metering {#api-metering}

The provider prices actual usage (fresh input, cached input, and
output tokens at the admitted model rates) using exact decimal
arithmetic. Let `prompt_tokens` be all input tokens and let
`prompt_tokens_details.cached_tokens` be the cached subset. Fresh
input tokens are their difference. The unrounded charge is:

~~~ text
fresh_input_tokens * input_rate
  + cached_tokens * cached_input_rate
  + completion_tokens * output_rate
~~~

The provider applies one ceiling per request: the debit is the
smallest whole number of credits not less than this sum. It reports
that integer in the usage object:

~~~ json
"usage": {
  "prompt_tokens": 1004,
  "prompt_tokens_details": { "cached_tokens": 192 },
  "completion_tokens": 214,
  "credits_charged": 1930
}
~~~

`credits_charged` is the TPX usage-accounting member, unchanged from
TPX v0.2: the total credits debited for the request, a non-negative
JSON integer in the range defined by {{credit}}. Debits are whole
credits, so the currency view is exact to six decimal places.

Before inference begins, the provider MUST determine a finite maximum
output-token count from the request or its documented default,
calculate a conservative maximum charge at the admitted rates with
the same ceiling, and atomically reserve that amount against both the
grant and the bound person balance ({{balances}}). If the reservation
cannot be made, it refuses before inference. On completion it commits the
actual rounded charge once and releases the rest. This is the TPX-A
reservation strategy required by AAuth-Budget's hard-cap invariant.

Streaming responses report `usage`, including `credits_charged`, in
the final SSE chunk. Once inference has begun, processed input and
generated output are chargeable even if the connection ends before
the final chunk; the grant-state endpoint is authoritative after an
ambiguous transport failure. Each accepted POST is a distinct
chargeable operation. TPX-A defines no idempotent replay mechanism,
so an agent MUST NOT automatically retry an ambiguous request without
first checking grant state.

## Grant State {#api-grant}

`GET /grant` returns the AAuth-Budget budget state, extended with
the credit-denominated view:

~~~ json
{
  "active": true,
  "budget": { "amount": "0.05", "currency": "USD" },
  "spent": { "amount": "0.04125", "currency": "USD" },
  "credits": 50000,
  "credits_used": 41250
}
~~~

`credits`:
: REQUIRED. The granted cap in credits, equal to the exact conversion
  of `budget.amount`.

`credits_used`:
: REQUIRED. Cumulative committed debits in credits, equal to the exact
  conversion of `spent.amount`. It excludes outstanding reservations.

Both members are non-negative JSON integers. The conversion is exact,
so the two views MUST NOT disagree. Remaining budget is the
difference. The response carries no identity and is not cacheable,
per AAuth-Budget. Agents SHOULD check it before large jobs, while
recognizing that concurrent requests can consume the snapshot's
remainder.

## Error Signals {#api-errors}

Token problems use the AAuth challenge; spend problems use
application-layer OpenAI-compatible error bodies:

| Status | Signal | Meaning | Recovery |
|---|---|---|---|
| 401 | `AAuth-Requirement` challenge | Auth token expired, invalid, or revoked | Obtain the new resource token and re-authorize through the PS; stop on `mission_terminated` |
| 402 | `error.code: "budget_exhausted"` | Remainder cannot cover the request's reservation | Retry with a smaller finite output bound, or request a new mission |
| 402 | `error.code: "balance_exhausted"` | Person's provider balance empty | The person tops off at the provider |
| 403 | `error.code: "model_not_allowed"` | Requested model is outside `budget.models` | Choose a granted model, or request a new mission |

`budget_exhausted` is AAuth-Budget's code carried in the
OpenAI-compatible nested body:

~~~ json
{
  "error": {
    "code": "budget_exhausted",
    "message": "This grant cannot cover the requested maximum cost.",
    "type": "invalid_request_error"
  }
}
~~~

A terminated mission surfaces as the 401 challenge at the provider
and as `mission_terminated` at the PS. Provider-side token or account
revocation surfaces as 401 followed by refusal at the Access Server.
TPX-A defines no separate application-layer revocation status.

# Balances and Account Binding {#balances}

The provider's Access Server is the account authority. During AAuth
federation it maps the PS assertion to the provider account of record
and keeps that mapping internal. When it needs a person identifier for
that lookup, it uses AAuth's `requirement=claims` exchange with the PS;
claims supplied on that protected PS-to-AS leg MUST NOT be copied into
the auth token. If no binding exists, the Access Server uses AAuth's
interaction or payment requirements to establish one before issuing
an auth token.

The provider MUST reserve and commit a debit atomically against both
the mission grant and the bound person balance; failure to reserve
either side produces no debit on the other. A provider MUST debit only
a balance bound through the asserting PS and MUST NOT accept an
account selector from the agent. How the person links a PS to an
existing account, tops up, or settles payment is provider UI and
payment-protocol surface outside this profile.

# Identity Minimization {#identity}

Grants convey tokens, not identity:

- Auth tokens MUST carry `inference` among their space-separated
  `scope` values, satisfying AAuth's requirement for at least one of
  `sub` or `scope`, and MUST omit `sub`.
- Auth tokens MUST NOT carry `email`, `name`, `tenant`, `groups`,
  `roles`, or any other person or account identity claim.
- Providers MUST NOT expose a stable cross-grant person identifier
  to agents in API responses, errors, or grant state. The agent can
  correlate requests only within a single mission reference, by
  design.

The Access Server can still identify the account while evaluating the
federation request and can keep an internal grant-to-account mapping;
that identifier does not need to appear in the auth token delivered
through the agent.

# Worked Example {#example}

Pony Chat proposes a 100000-credit ($0.10) grant; the person
approves 50000 credits ($0.05). The approved mission blob:

~~~ json
{
  "approver": "https://ps.example",
  "agent": "aauth:pony-chat@ponychat.tokenpony.dev",
  "approved_at": "2026-07-18T14:32:11Z",
  "description":
    "Chat in Pony Chat using your Token Pony balance.",
  "approved_tools": [
    { "name": "chat.completions",
      "description":
        "OpenAI-compatible chat completions at api.tokenpony.dev" }
  ],
  "capabilities": ["interaction"],
  "budgets": [
    { "resource": "https://api.tokenpony.dev",
      "amount": "0.05",
      "currency": "USD",
      "models": ["pony-8b", "pony-70b"] }
  ]
}
~~~

Per AAuth, `s256` is computed over the exact response body bytes;
for the compact (whitespace-free) serialization of this blob, in the
member order shown, the mission reference is:

~~~ text
AAuth-Mission: approver="https://ps.example";
    s256="EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
~~~

The agent reads the granted values from the blob: it asked for
$0.10 and holds $0.05. An auth token issued under the mission:

~~~ json
{
  "iss": "https://as.tokenpony.dev",
  "dwk": "aauth-access.json",
  "aud": "https://api.tokenpony.dev",
  "scope": "inference",
  "agent": "aauth:pony-chat@ponychat.tokenpony.dev",
  "cnf": { "jwk": { "kty": "OKP", "crv": "Ed25519", "x": "..." } },
  "jti": "at_9Kp2vN7sR1tY8mZ3qX5b",
  "iat": 1784386800,
  "exp": 1784390400,
  "mission": {
    "approver": "https://ps.example",
    "s256": "EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
  },
  "budget": {
    "resource": "https://api.tokenpony.dev",
    "amount": "0.05",
    "currency": "USD",
    "models": ["pony-8b", "pony-70b"]
  }
}
~~~

The `budget` claim is the granted entry, verbatim, per AAuth-Budget;
the provider learns the cap from the token and needs no call to the
PS. After 41250 credits of chat, `GET /grant` returns the state
shown in {{api-grant}}, and a request that would exceed the
remainder fails with `budget_exhausted` ({{api-errors}}). {{e2e}}
walks the full exchange.

# Conformance {#conformance}

An implementation conforms in one of three roles, each on top of the
corresponding AAuth-Budget roles.

A **TPX-A provider** operates a budgeted resource and a budget-aware
AAuth Access Server, and additionally:

- prices in credits at the fixed conversion of {{credit}} and
  publishes exact rates at `GET /v1/models`;
- enforces the `models` restriction ({{models}});
- serves the endpoints of {{api-endpoints}}, reporting
  `credits_charged` in every usage object and the credit view at
  `GET /grant`;
- reserves and commits against the grant and balance atomically, and
  computes charges and rounding per {{api-metering}};
- signals errors per {{api-errors}};
- binds grants to balances per {{balances}}; and
- keeps identity out of tokens, grant state, and usage
  ({{identity}}).

A **TPX-A agent** is a budgeted agent that additionally:

- handles a 401 challenge by obtaining its new resource token and
  re-authorizing through the PS, stops on `mission_terminated`, and
  surfaces 402 to the person; and
- SHOULD check `GET /grant` before large jobs and propose
  completion, with total spend, when the task is done or the budget
  is exhausted.

A **Person Server** conforms per AAuth-Budget, including use of the
`budget` federation parameter. TPX-A adds the `models` member to what
it must recognize, attenuate, and render ({{models}}).

# Security Considerations

AAuth-Budget's security considerations apply: the cap is the
backstop behind proof of possession, the meter is the payee, and
consent is the ceiling. TPX-A adds:

**Consent phishing.** TPX v0.2's open registration meant
self-asserted app names, mitigated by verified origins. The TPX-A
agent identifier is domain-verified by construction. The residual
surface is the mission description: attacker-influenceable text the
PS MUST sanitize and keep visually distinct from the amount, per
AAuth and AAuth-Budget.

**The Person Server as trusted party.** The person chooses the PS,
and the provider debits balances only for grants asserted by the PS
bound to the account ({{balances}}). A compromised PS can therefore
approve missions against the balances of persons who linked it, up
to each consent-rendered amount per mission. Providers SHOULD
rate-limit issuance per PS and surface per-grant spend in their
dashboard.

**Price changes.** A budget caps money, not tokens or model quality.
A provider can consume the cap faster by raising its published rates.
Agents SHOULD fetch `GET /v1/models` before large jobs, and providers
MUST snapshot the admitted rate for each request as specified in
{{api-metering}}. This makes a request auditable but does not make the
payee's pricing trustworthy.

**Storage.** There are no shared client secrets or bearer
credentials. The agent's signing key remains sensitive keying
material. Auth tokens are proof-of-possession tokens and are useless
without that key, per AAuth.
Providers persist the meter and account bindings; the mission blob
is held by the agent and the PS, byte-exact under `s256`, so
after-the-fact alteration of a grant is detectable by any holder of
the original bytes.

# Privacy Considerations

What the provider learns (usage per grant per agent) is inherent to
metering, as in TPX v0.2, and {{identity}} bounds it there: no
person identity claims and no cross-grant correlation handle for
agents. The provider's Access Server necessarily retains the internal
account binding.

What the PS learns is new relative to TPX v0.2: the person's own
chosen governor sees mission text and which agents use which
providers. The provider never sees the description; the blob stays
with the agent and the PS, per AAuth.

# IANA Considerations {#iana}

This document has no IANA actions. The `models` member rides inside
a `budgets` entry, whose profile extensibility
{{I-D.draft-mcguinness-aauth-budget}} states; `credits`,
`credits_used`, and `credits_charged` are members of API-surface
responses this profile defines; and `balance_exhausted` and
`model_not_allowed` are API-surface error codes.

--- back

# Mapping from TPX v0.2 {#mapping}

| TPX v0.2 (OAuth 2.0) | TPX-A (AAuth) |
|---|---|
| RFC 9728 + RFC 8414 discovery | AAuth resource and person metadata, challenge-first |
| RFC 7591 registration; client ID metadata documents | None: agent identity by construction (agent token) |
| Public/confidential clients; hashed shared secrets | Dissolved: no shared client or provider credential; the agent protects only its own signing key |
| Authorization endpoint, redirect, code, iss (RFC 9207) | Mission proposal, clarification, approval at the PS; no front channel |
| PKCE (RFC 7636); state | Retired with the front channel |
| PAR (RFC 9126) | The proposal is already back-channel |
| RAR type llm-inference (RFC 9396) | AAuth-Budget budgets entry + TPX-A models |
| Resource indicator (RFC 8707) | The entry's resource + resource-token audience |
| Refresh token as the grant; rotation, reuse detection | The mission is the grant; nothing rotates |
| Access token, expires_in <= 3600 | Auth token, exp <= 1 hour, key-bound |
| DPoP optional (RFC 9449) | Proof of possession is the substrate: cnf.jwk + signed requests on every call |
| Introspection (RFC 7662) with budget_used | Signed `GET /grant` with `spent` and `credits_used` |
| Revocation (RFC 7009); provider dashboard | Mission revocation at the PS; the reissue gate bounds latency to one token lifetime |
| Budget consent at the provider | Budget consent at the person's PS; provider interaction is limited to account binding or payment |
| Budget as integer credits on the wire | Budget as currency, committed; credits at the API surface, exactly convertible |
| credits_charged in usage | Unchanged |
| 401 invalid_token / 402 spend signals | Same meanings; the 401 challenge is AAuth-Requirement |

# Complete Protocol Exchange {#e2e}

This appendix is non-normative. It walks Pony Chat through one full
grant: rates, proposal, approval, challenge, issuance, inference,
state, exhaustion, and completion. JWTs are abbreviated, signatures
are elided, and AAuth's interaction machinery is reduced to its
shape; AAuth's own examples govern the substrate details. Later
messages also elide `Signature-Input` and `Signature`; every request
remains signed as the first ones show.

Anyone can read the menu. The provider's rates, in credits per
token:

~~~ http-message
GET /v1/models HTTP/1.1
Host: api.tokenpony.dev
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{
  "object": "list",
  "data": [
    { "id": "pony-8b", "object": "model",
      "credits_per_token":
        { "input": "1", "cached_input": "0.25", "output": "5" } },
    { "id": "pony-70b", "object": "model",
      "credits_per_token":
        { "input": "4", "cached_input": "1", "output": "20" } }
  ]
}
~~~

The agent `aauth:pony-chat@ponychat.tokenpony.dev` proposes a
mission at the person's PS, asking for 100000 credits ($0.10):

~~~ http-message
POST /mission HTTP/1.1
Host: ps.example
Content-Type: application/json
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key");created=1784384400
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "description":
    "Chat in Pony Chat using your Token Pony balance.",
  "tools": [
    { "name": "chat.completions",
      "description":
        "OpenAI-compatible chat completions at api.tokenpony.dev" }
  ],
  "budgets": [
    { "resource": "https://api.tokenpony.dev",
      "amount": "0.10",
      "currency": "USD",
      "models": ["pony-8b", "pony-70b"] }
  ]
}
~~~

The PS defers while the person reviews (202, `status: pending`),
then the person grants half the ask. Polling returns the approved
mission of {{example}}, byte for byte, with its reference:

~~~ http-message
HTTP/1.1 200 OK
AAuth-Mission: approver="https://ps.example";
    s256="EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
Content-Type: application/json

{
  "approver": "https://ps.example",
  "agent": "aauth:pony-chat@ponychat.tokenpony.dev",
  "approved_at": "2026-07-18T14:32:11Z",
  ...
  "budgets": [
    { "resource": "https://api.tokenpony.dev",
      "amount": "0.05",
      "currency": "USD",
      "models": ["pony-8b", "pony-70b"] }
  ]
}
~~~

The agent reads the granted values from the blob: 50000 credits.
Its first inference attempt, signed with its agent token, draws the
challenge and a resource token:

~~~ http-message
POST /v1/chat/completions HTTP/1.1
Host: api.tokenpony.dev
Content-Type: application/json
AAuth-Mission: approver="https://ps.example";
    s256="EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key" "aauth-mission");created=1784386740
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."
~~~

~~~ http-message
HTTP/1.1 401 Unauthorized
AAuth-Requirement: requirement=auth-token;
    resource-token="eyJ..resource-token.."
~~~

The resource token copies the mission reference from the header.
Its payload:

~~~ json
{
  "iss": "https://api.tokenpony.dev",
  "dwk": "aauth-resource.json",
  "aud": "https://as.tokenpony.dev",
  "agent": "aauth:pony-chat@ponychat.tokenpony.dev",
  "agent_jkt": "kV9CqXP3...",
  "scope": "inference",
  "mission": {
    "approver": "https://ps.example",
    "s256": "EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
  },
  "jti": "rt_8Zw2mKq4",
  "iat": 1784386740,
  "exp": 1784387040
}
~~~

The agent presents it at the PS, which verifies that the mission is
active and the issuer matches the granted entry:

~~~ http-message
POST /token HTTP/1.1
Host: ps.example
Content-Type: application/json
AAuth-Mission: approver="https://ps.example";
    s256="EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key" "aauth-mission");created=1784386790
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "resource_token": "eyJ..resource-token..",
  "justification": "Answer the user's next message in Pony Chat."
}
~~~

Because TPX-A uses federated access, the PS then sends the resource
token, agent token, and granted entry to the provider's Access Server.
The `budget` value comes from the approved blob, not the agent's
request:

~~~ http-message
POST /token HTTP/1.1
Host: as.tokenpony.dev
Content-Type: application/json
Signature-Key: sig=jwks_uri;
    jwks_uri="https://ps.example/.well-known/jwks.json"

{
  "resource_token": "eyJ..resource-token..",
  "agent_token": "eyJ..agent-token..",
  "budget": {
    "resource": "https://api.tokenpony.dev",
    "amount": "0.05",
    "currency": "USD",
    "models": ["pony-8b", "pony-70b"]
  }
}
~~~

The Access Server uses an existing PS-to-account binding (otherwise it
would run AAuth's claims and interaction requirements), checks that
`resource_token.iss`, the future auth-token audience, and
`budget.resource` are identical, and mints the auth token of
{{example}}. Its response to the PS:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{ "auth_token": "eyJ..auth-token..", "expires_in": 3600 }
~~~

The PS verifies that token and returns it to the agent:

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{ "auth_token": "eyJ..auth-token..", "expires_in": 3600 }
~~~

Inference proper, signed with the key the auth token's `cnf.jwk`
names:

~~~ http-message
POST /v1/chat/completions HTTP/1.1
Host: api.tokenpony.dev
Content-Type: application/json
Signature-Key: sig=jwt;jwt="eyJ..auth-token.."

{
  "model": "pony-8b",
  "messages": [ { "role": "user", "content": "..." } ],
  "max_completion_tokens": 256,
  "stream": false
}
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "chatcmpl-7QkX",
  "model": "pony-8b",
  "choices": [
    { "message": { "role": "assistant", "content": "..." },
      "finish_reason": "stop" }
  ],
  "usage": {
    "prompt_tokens": 1004,
    "prompt_tokens_details": { "cached_tokens": 192 },
    "completion_tokens": 214,
    "credits_charged": 1930
  }
}
~~~

At pony-8b rates that is 812 fresh-input credits, 48 cached-input
credits, and 1070 output credits: 1930, already a whole number, so
1930 is committed and the unused reservation is released. Sessions
later, after replacing expired auth tokens by obtaining fresh resource
tokens and repeating the federation exchange, the agent checks the
meter:

~~~ http-message
GET /grant HTTP/1.1
Host: api.tokenpony.dev
Signature-Key: sig=jwt;jwt="eyJ..auth-token.."
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "active": true,
  "budget": { "amount": "0.05", "currency": "USD" },
  "spent": { "amount": "0.04125", "currency": "USD" },
  "credits": 50000,
  "credits_used": 41250
}
~~~

8750 credits remain. When the remainder can no longer cover a
request's maximum charge, the completion fails closed in the API's
own error shape:

~~~ http-message
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "error": {
    "code": "budget_exhausted",
    "message": "This grant cannot cover the requested maximum cost.",
    "type": "invalid_request_error"
  }
}
~~~

The agent surfaces the exhaustion to the person and proposes
completion, spend in the summary:

~~~ http-message
POST /interaction HTTP/1.1
Host: ps.example
Content-Type: application/json
AAuth-Mission: approver="https://ps.example";
    s256="EMIlPYHAY6dNVw_YguH7Vqde9hCAAtLHWRtZfCndpUc"
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "type": "completion",
  "summary":
    "Chat ended. Spent 50000 of 50000 credits ($0.05)."
}
~~~

If the person wants more Pony Chat, that is a new mission with a
fresh budget; the person decides, exactly as in TPX v0.2's top-off.
Any later token request under the terminated reference fails
closed:

~~~ json
{
  "error": "mission_terminated",
  "error_description": "This mission has ended."
}
~~~

# Acknowledgments
{:numbered="false"}

TPX v0.2 {{TPX}} defined the metered LLM inference grant as an OAuth
2.0 profile. This document restates it natively on AAuth via
Budgeted Missions for AAuth, which generalizes its budget mechanism.
