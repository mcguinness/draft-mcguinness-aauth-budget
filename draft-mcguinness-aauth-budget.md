---
title: "Budgeted Missions for AAuth"
abbrev: "AAuth Budget"
category: std

docname: draft-mcguinness-aauth-budget-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
keyword:
 - aauth
 - agent
 - budget
 - spend cap
 - metering
 - authorization
venue:
  github: "mcguinness/draft-mcguinness-aauth-budget"
  latest: "https://mcguinness.github.io/draft-mcguinness-aauth-budget/draft-mcguinness-aauth-budget.html"

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

informative:
  RFC9421:
  TPX:
    title: "TPX v0.2: An OAuth 2.0 profile for metered LLM inference grants"
    target: https://tokenpony.dev/spec/
    date: 2026
  I-D.draft-mcguinness-aauth-budget-tpx:
    title: "TPX-A: An AAuth Profile for Metered LLM Inference Grants"
    target: https://mcguinness.github.io/draft-mcguinness-aauth-budget/draft-mcguinness-aauth-budget-tpx.html
    author:
      -
        ins: K. McGuinness
        name: Karl McGuinness
    date: 2026
  I-D.draft-mcguinness-mission-aauth:
    title: "Mission-Bound Authorization for AAuth"
    target: https://mcguinness.github.io/mission-bound-authorization/draft-mcguinness-mission-aauth.html
    author:
      -
        ins: K. McGuinness
        name: Karl McGuinness
    date: 2026
  I-D.draft-mcguinness-mission-metering:
    title: "Mission Consumption Metering"
    target: https://mcguinness.github.io/mission-bound-authorization/draft-mcguinness-mission-metering.html
    author:
      -
        ins: K. McGuinness
        name: Karl McGuinness
    date: 2026

--- abstract

AAuth missions express intent and tools, never quantity: nothing a
person approves says how much an agent may spend. This document
defines the budget, a hard cap on cumulative monetary spend at one
resource, proposed by the agent as part of its mission, approved by
the person at the Person Server, committed under the mission's s256,
carried in every auth token, and enforced by the resource that meters
its own service. A budget is a damage cap, not a payment instrument:
a compromised agent spends at most the remainder. The extension adds
one mission member, one auth-token claim, one resource metadata
member, and one error code, and no new protocol.

--- middle

# Introduction

The AAuth protocol {{I-D.draft-hardt-oauth-aauth-protocol}} gives
agents their own identity, routes their authorization through a
Person Server (PS), and binds approved missions to requests by
reference. Its missions express intent and tools, never quantity.
AAuth reserves HTTP status 402 for the case where payment is
additionally required, but defines no authorization object behind it:
nothing a person approves says how much an agent may spend.

This document supplies that object. A budget is a hard cap on
cumulative spend at one resource:

- proposed by the agent in its mission proposal;
- approved, possibly lowered, by the person at the PS;
- committed under the mission's `s256`, so the cap is immutable and
  the reference to it is signature-covered on every request;
- carried in every auth token issued under the mission, so the
  resource needs no call to the PS; and
- enforced by the resource, which meters its own service and refuses
  further spend once the cap is reached.

The budget is a damage cap, not a payment instrument. AAuth's
substrate already key-binds every token and signs every request; the
budget bounds the remaining case, agent or key compromise, at the
amount the person consented to.

The extension adds four names and no new protocol: one mission member
(`budgets`), one auth-token claim (`budget`), one resource metadata
member and endpoint (`budget_endpoint`), and one error code
(`budget_exhausted`).

Profiles pin what this document deliberately leaves open: the pricing
unit a resource publishes, the carriage of per-response debit
reporting, the error body of its API surface, and the payment
relationship behind the balance. TPX-A
({{I-D.draft-mcguinness-aauth-budget-tpx}}), the AAuth profile of the
Token Pony Express {{TPX}}, is the motivating profile: metered LLM
inference grants, with credits as the published unit and an
OpenAI-compatible API surface.

## Applicability

This document tracks draft-hardt-oauth-aauth-protocol-08, an
individual Internet-Draft; a change to AAuth's mission or token
surfaces revises this document. It requires a deployment that
operates a Person Server and uses AAuth missions.

# Conventions and Terminology

{::boilerplate bcp14-tagged}

This document uses Person Server (PS), Access Server, agent, agent
token, resource token, auth token, mission blob, mission log, `s256`,
and the `AAuth-Mission` header as defined by
{{I-D.draft-hardt-oauth-aauth-protocol}}. It additionally uses:

Budget:
: A hard cap on cumulative monetary spend at one resource under one
  mission, as this document defines it.

Payee:
: The resource being paid: the party that prices its own service,
  meters usage, and enforces the budget.

All JSON shown in this document is non-normative and illustrative;
the member definitions in the surrounding text are authoritative.

# Scope and Trust Boundary {#scope}

This is a payee-enforced damage cap, not a governance meter. The
meter lives at the resource because only the resource knows usage,
and the resource prices its own service. That self-reported meter is
sound precisely because the budget bounds what that one resource may
debit from a relationship the person already holds with it.

Consequently:

- A budget bounds spend at the named resource. It does not bound
  agent behavior across resources.
- Cross-resource aggregate spend, call-count caps, and duration caps
  are governance-layer consumption bounds with distributed-counting
  machinery of their own, and are out of scope. Mission governance
  profiles for AAuth Person Servers address that layer
  ({{I-D.draft-mcguinness-mission-aauth}},
  {{I-D.draft-mcguinness-mission-metering}}); this extension
  requires none of them and composes with them unchanged, since
  `budgets` is an ordinary blob member committed under `s256`.
- Non-monetary quantity caps are out of scope. Budgets are money:
  the one unit every person can consent to.

# The budgets Mission Member {#budgets}

An agent proposes budgets in its mission proposal; the approved
mission blob carries the granted values. Both use one member:

~~~ json
{
  "budgets": [
    { "resource": "https://api.search.example",
      "amount": "2.00",
      "currency": "USD" }
  ]
}
~~~

Each entry has the members:

`resource`:
: REQUIRED. The resource identifier the cap applies to, an absolute
  HTTPS URI. At most one entry per resource.

`amount`:
: REQUIRED. A string. A positive decimal number: the maximum
  cumulative spend.

`currency`:
: REQUIRED. A string. An ISO 4217 currency code.

A profile MAY add members to an entry. Consumers fail closed: a
Person Server MUST reject a proposal whose `budgets` entries carry
members it does not recognize or values it cannot render for
consent, and a resource MUST refuse budgeted access it cannot fully
enforce ({{claim}}).

## Consent {#consent}

The PS authenticates the person and renders each proposed entry: the
resource, and the amount with its currency. The person or the PS MAY
grant a lower amount than proposed; the granted amount MUST NOT
exceed the proposed amount. The blob's `budgets` member carries the
granted values, and the agent MUST read them from the blob, not from
its proposal.

The mission description is the agent's narrative, not the grant: the
PS MUST render it sanitized, per AAuth, and visually distinct from
the amounts it asks the person to approve.

## Immutability and Top-Off {#immutability}

The blob is immutable under `s256`, so a budget cannot be raised in
place. A top-off is a new mission proposal; the person decides.
AAuth's permission endpoint is not used for budget changes, because
the cap is committed under `s256`. When a budget is spent or the
task is done, the agent SHOULD propose completion with a summary
that includes total spend.

## Worked Example {#example}

An approved mission blob for a digest agent with a $2.00 cap:

~~~ json
{
  "approver": "https://ps.example",
  "agent": "aauth:scout@agents.example",
  "approved_at": "2026-07-18T09:05:42Z",
  "description":
    "Compile a daily digest of Doppler launch coverage this week.",
  "approved_tools": [
    { "name": "search.query",
      "description": "Web search at api.search.example" }
  ],
  "capabilities": ["interaction"],
  "budgets": [
    { "resource": "https://api.search.example",
      "amount": "2.00",
      "currency": "USD" }
  ]
}
~~~

Per AAuth, `s256` is computed over the exact response body bytes;
for the compact (whitespace-free) serialization of this blob, in the
member order shown, the mission reference is:

~~~ text
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
~~~

The reference is covered by the HTTP Message Signature {{RFC9421}}
on every request that carries it, per AAuth, so the cap the person
approved is bound to every call made under it.

# The budget Auth-Token Claim {#claim}

An auth token issued under a mission whose `budgets` member has an
entry matching the token's audience MUST carry that granted entry,
verbatim, in a `budget` claim, and only that entry:

~~~ json
{
  "iss": "https://ps.example",
  "dwk": "aauth-person.json",
  "aud": "https://api.search.example",
  "sub": "p-2c9wqe",
  "agent": "aauth:scout@agents.example",
  "cnf": { "jwk": { "kty": "OKP", "crv": "Ed25519", "x": "..." } },
  "jti": "at_5Xr8kQ2mVn3pY7wZ1sB4",
  "iat": 1784386800,
  "exp": 1784390400,
  "mission": {
    "approver": "https://ps.example",
    "s256": "q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
  },
  "budget": {
    "resource": "https://api.search.example",
    "amount": "2.00",
    "currency": "USD"
  }
}
~~~

- The entry's `resource` MUST equal the token's `aud`.
- The `mission` claim is AAuth's own. The resource never
  dereferences the blob, so the cap travels in the token and the
  resource needs no call to the PS.
- Before issuing, the PS MUST verify that the mission is active and
  that the resource token's issuer matches a granted entry's
  `resource`. In AAuth's federated mode the PS conveys the granted
  entry in its federation request, and the Access Server MUST copy
  it into the auth token it mints.

Auth tokens remain proof-of-possession and short-lived per AAuth, so
every renewal repasses the PS gate: a revoked mission stops new
spend authority within one auth-token lifetime.

# Metering {#metering}

The resource is the meter. It MUST:

- key the meter by the mission reference (`approver`, `s256`): the
  cap is cumulative across all auth tokens and renewals under the
  mission, not per token;
- debit atomically with serving each request, in the committed
  currency (a resource that prices in its own units converts at its
  published relationship, a profile concern); and
- refuse further budgeted requests once cumulative debits reach the
  granted amount.

A request whose worst-case cost cannot fit the remainder MAY be
refused up front. Exact concurrency control (reserve and commit) is
out of scope: a resource that admits requests concurrently accepts a
race bounded by the cost of requests in flight, or refuses up front.

A resource SHOULD report each debit in its response; the carriage is
API surface, pinned by profiles.

# Budget State {#state}

A resource that enforces budgets MUST publish a `budget_endpoint`
member in its AAuth resource metadata document. The endpoint,
authenticated like any resource request (signed, with the auth token
and the mission reference), returns the state of the budget named by
the presented mission reference:

~~~ json
{
  "active": true,
  "budget": { "amount": "2.00", "currency": "USD" },
  "spent": { "amount": "0.31", "currency": "USD" }
}
~~~

`active`:
: REQUIRED. A boolean. Whether the resource still serves budgeted
  requests under this mission.

`budget`:
: REQUIRED. The granted cap, as committed: an object with `amount`
  and `currency` as defined in {{budgets}}.

`spent`:
: REQUIRED. Cumulative debits to date, in the same shape. Remaining
  budget is the difference.

The response MUST NOT include `sub` or any other identity claim.
Agents SHOULD read it before starting work whose cost is significant
relative to the remainder.

# Exhaustion {#exhaustion}

When the cap is reached, the resource refuses with HTTP status 402
and the error code `budget_exhausted`:

~~~ json
{
  "error": "budget_exhausted",
  "error_description":
    "The mission's budget at this resource is spent."
}
~~~

The body shown is AAuth's flat error shape; a profile MAY pin its
API's native error carriage instead. The code, and the meaning "the
committed cap is spent, a new mission is the recovery", are this
document's.

Payment failures outside the cap (an empty prepaid balance, a failed
settlement) belong to the resource's payment relationship with the
person. They are signaled with status 402 per AAuth and named by the
profile, not by this document.

# Conformance {#conformance}

An implementation conforms in one of three roles.

A **budgeted resource**:

- publishes `budget_endpoint` in its resource metadata and serves it
  per {{state}};
- rejects budgeted access on tokens lacking the `mission` or
  `budget` claims, or carrying values it cannot enforce, failing
  closed ({{claim}});
- meters per {{metering}} and signals exhaustion per {{exhaustion}};
  and
- keeps identity out of budget-state responses.

A **budgeted agent**:

- proposes caps via `budgets` and reads the granted values from the
  approved blob ({{consent}});
- carries the mission reference on every request under the grant,
  signature-covered, per AAuth; and
- stops spending against an exhausted budget: recovery is a new
  mission, decided by the person ({{immutability}}).

A **Person Server**:

- renders consent per {{consent}} and supports granting lower
  amounts;
- returns the granted entries in the approved blob;
- gates issuance on mission state and on the resource matching a
  granted entry ({{claim}}); and
- carries the `budget` claim in auth tokens it issues, or conveys
  the granted entry in federation.

# Security Considerations

**The cap is the backstop, not the lock.** AAuth's substrate already
key-binds every token and signs every request; a leaked auth token
spends nothing without the agent's key. The budget bounds the
remaining case, agent or key compromise, at the amount the person
consented to.

**The meter is the payee.** A resource can misreport its own meter
exactly as any merchant can misprice. The budget-state endpoint and
per-response debit reporting keep the meter auditable by the paying
person, and the PS's mission log records every token request under
the grant for reconciliation. This document does not attempt to make
the payee's meter trustless.

**Consent is the ceiling.** The granted amount never exceeds the
proposed amount, the blob is immutable under `s256`, and the
reference is signature-covered per request: what the person approved
is what every call is bound to, byte-exact. A PS that cannot render
a proposed entry rejects the proposal rather than approving what the
person could not see ({{budgets}}).

**Issuance is the gate.** Every auth token under a mission passes
the PS, and tokens are short-lived per AAuth, so revocation latency
is bounded by the auth-token lifetime even where a resource retains
no revocation surface.

# Privacy Considerations

**Each resource sees only its own entry.** The `budget` claim
carries the single entry matching the token's audience, so a
resource never learns the person's caps at other resources. The full
`budgets` member travels only in the blob, which stays with the
agent and the PS per AAuth.

**Budget state carries no identity.** The budget-state response is
keyed by the mission reference and returns amounts only. What the
resource learns, spend per mission per agent, is inherent to
metering; the mission reference correlation it rides is AAuth's own,
unchanged by this document.

**Amounts are sensitive.** A granted cap reveals what the person was
willing to spend. It is disclosed to exactly the parties that must
enforce or audit it: the PS, the agent, and the named resource.

# IANA Considerations {#iana}

This document has no IANA actions. The members it defines ride
inside structures whose extensibility their defining specification
states: `budgets` in the PS-produced mission blob, `budget` in the
auth token, and `budget_endpoint` in the resource metadata document,
whose unrecognized members AAuth consumers ignore. Should AAuth
establish registries for those structures, the members this document
defines would be registered there.

--- back

# Acknowledgments
{:numbered="false"}

This extension generalizes the budget mechanism of the Token Pony
Express {{TPX}}, which demonstrated the budget-as-grant pattern as an
OAuth 2.0 profile, onto AAuth's mission substrate. TPX-A
({{I-D.draft-mcguinness-aauth-budget-tpx}}) profiles this document
back onto that use case.
