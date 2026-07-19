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
  RFC7519:
  ISO4217:
    title: "ISO 4217:2015, Codes for the representation of currencies"
    target: https://www.iso.org/standard/64758.html
    author:
      -
        org: International Organization for Standardization
    date: 2015

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

--- abstract

AAuth missions express intent and tools, never quantity: nothing a
person approves says how much an agent may spend. This document
defines the budget, a hard cap on cumulative monetary spend at one
resource, proposed by the agent as part of its mission, approved by
the person at the Person Server, committed under the mission's s256,
carried in every applicable auth token, and enforced by the resource
that meters its own service. A budget is a damage cap, not a payment
instrument: a compromised agent spends at most the remainder. The
extension adds one mission member, one auth-token claim (also used as
a federation request parameter), one resource metadata member, and
one error code on existing AAuth surfaces.

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
  the mission reference is issuer-signed in the auth token presented
  on each budgeted request;
- carried in every auth token issued under the mission for that
  resource, so the resource needs no call to the PS; and
- enforced by the resource, which meters its own service and refuses
  further spend once the cap is reached.

The budget is a damage cap, not a payment instrument. AAuth's
substrate already key-binds every token and signs every request; the
budget bounds the remaining case, agent or key compromise, at the
amount the person consented to.

The extension adds four names on existing AAuth surfaces: one mission
member (`budgets`), one auth-token claim also used in PS-to-AS
federation (`budget`), one resource metadata member and endpoint
(`budget_endpoint`), and one error code (`budget_exhausted`).

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

Committed debit:
: A charge durably applied to the meter for a served operation.

Reservation:
: An amount temporarily held against a budget before or while an
  operation is served, to prevent concurrent operations from
  exceeding the cap.

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
  are governance-layer consumption bounds: metering them exactly
  across decision points is a distributed-counting problem with
  reserve, commit, and settlement machinery of its own, and is out
  of scope. Governance profiles that meter at a policy decision
  point compose with this extension unchanged, since `budgets` is an
  ordinary blob member committed under `s256`.
- Non-monetary quantity caps are out of scope. Budgets are money:
  the one unit every person can consent to.
- The cap is an accounting invariant, including under concurrency and
  failure recovery. A deployment that permits in-flight operations to
  overshoot is not conformant to this document.

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
  HTTPS URL conforming to AAuth's Server Identifier requirements. It
  is compared by exact string match. At most one entry per resource.

`amount`:
: REQUIRED. The maximum cumulative spend: a decimal string with a
  positive value. A decimal string is one or more ASCII digits,
  optionally followed by a period and one or more ASCII digits, with
  no leading zero in the integer part unless it is exactly `0`, and
  no sign, exponent, grouping character, or whitespace. The grammar
  admits zero; an `amount` MUST be positive.

`currency`:
: REQUIRED. A three-uppercase-ASCII-letter currency code registered
  by ISO 4217 {{ISO4217}}. The special-purpose codes `XTS` and `XXX`
  MUST NOT be used.

A present `budgets` member MUST be a non-empty array; entries for
different resources are independent. Amounts are compared
numerically in exact base-10 arithmetic — never lexically or in
binary floating point that could overshoot the cap. Profiles MAY
limit the supported number of digits or fractional digits.

The proposal is a request, never authority: the granted values exist
only in the approved blob. Relative to the proposal, every granted
entry:

- MUST match a proposed entry by exact `resource` comparison: the PS
  MAY omit a proposed entry but MUST NOT add one for an unproposed
  resource;
- MUST retain the proposed `currency`; and
- MUST NOT exceed the proposed `amount`.

A profile MAY add members to an entry; one that lets the PS change
an added member MUST define which changes are attenuating, and a
granted value MUST NOT broaden the proposed authority. Consumers
fail closed:

- a PS MUST reject a proposal whose entries carry members it does
  not recognize or values it cannot render for consent; and
- a resource MUST refuse budgeted access it cannot fully enforce
  ({{claim}}).

## Consent {#consent}

The PS authenticates the person and renders each proposed entry: the
resource, and the amount with its currency. The person or the PS MAY
omit an entry or grant a lower amount than proposed, subject to the
attenuation rules in {{budgets}}. The blob's `budgets` member carries
the granted values, and the agent MUST read them from the blob, not
from its proposal.

The mission description is the agent's narrative, not the grant: the
PS MUST render it sanitized, per AAuth, and visually distinct from
the amounts it asks the person to approve.

## Immutability and Top-Off {#immutability}

The blob is immutable under `s256`, so a budget cannot be raised in
place. A top-off is a new mission proposal; the person decides.
AAuth's permission endpoint is not used for budget changes, because
the cap is committed under `s256`. A budget has no lifetime of its
own: it ends with the mission. When a budget is spent or the task is
done, the agent SHOULD propose completion with a summary that
includes total spend.

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

On initial access, the reference is carried in the `AAuth-Mission`
header and covered by the HTTP Message Signature {{RFC9421}}. On
budgeted access, the same reference and the cap are issuer-signed in
the auth token, whose `Signature-Key` field is covered by the request
signature. {{e2e}} walks this grant end to end.

# The budget Auth-Token Claim {#claim}

An auth token issued under a mission whose `budgets` member has an
entry matching the token's audience MUST carry that granted entry,
verbatim, in a `budget` claim, and only that entry:

~~~ json
{
  "iss": "https://ps.example",
  "dwk": "aauth-person.json",
  "aud": "https://api.search.example",
  "scope": "search",
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

- The entry's `resource` MUST equal the token's `aud`, using exact
  string comparison.
- The `mission` claim is AAuth's own; the cap travels in the token,
  so the resource never dereferences the blob or calls the PS.
- Before issuing a token that carries a `budget` claim, the PS MUST
  verify that the mission is active and that the resource token's
  issuer equals the entry's `resource`, using exact string
  comparison.

In AAuth's federated mode, the budget travels as a `budget`
parameter on the authenticated PS-to-AS token request; an agent
cannot supply or modify it. The PS:

- MUST include the parameter, holding the granted entry verbatim,
  whenever it federates a token request for a resource with a
  granted entry; and
- MUST select the entry using the resource token's `mission` and
  `iss` claims, never a value from the agent's token request. The
  parameter asserts the granted entry for the mission named by the
  accompanying resource token's `mission` claim.

The Access Server:

- MUST verify that the resource token carries a `mission` claim, and
  copy it into the auth token it mints;
- MUST verify that the resource token's `iss` and the minted token's
  `aud` both equal `budget.resource`; and
- MUST copy the parameter verbatim into the auth token's `budget`
  claim once the checks pass, and refuse issuance when the parameter
  is malformed, cannot be enforced, or is absent where its policy
  requires a budget.

The Access Server never sees the blob, so it cannot detect an
omitted parameter; the resulting token simply carries no `budget`
claim, and the resource fails closed.

A token issued under the mission for a resource with no entry
carries no `budget` claim and conveys no budgeted access; whatever
else it conveys is ordinary AAuth authorization, outside this
document.

A resource MUST take the cap and mission reference only from a
verified auth token, never from request content or any other source,
and MUST reject a request whose `AAuth-Mission` header differs from
the token's `mission` claim.

Auth tokens remain proof-of-possession and short-lived per AAuth.
Re-authorization obtains a fresh resource token and repasses the PS
gate (and the Access Server in federated mode), so a terminated
mission cannot create new spend authority.

# Metering {#metering}

The resource is the meter. It MUST:

- key the meter by the mission reference (`approver`, `s256`): the
  cap is cumulative across all auth tokens and renewals under the
  mission, not per token;
- durably bind that meter to the complete `budget` value from the
  first accepted auth token and reject any later token under the same
  mission reference whose `budget` value is not identical;
- account in the committed currency, using exact arithmetic (a
  resource that prices in its own units converts at its published
  relationship, a profile concern);
- durably and atomically maintain the invariant that committed debits
  plus outstanding reservations never exceed the granted amount; and
- refuse or bound any operation that cannot be served while
  preserving that invariant.

Two `budget` objects are identical when they have the same member
names and recursively equal JSON values: member order is irrelevant,
array order is significant, strings compare exactly. A resource MUST
NOT replace the bound value even with a numerically equivalent or
lower amount.

Before an operation can incur cost, the resource MUST bound it with
one of:

- an atomic reservation of the operation's maximum cost;
- an atomic check before each chargeable unit; or
- another mechanism with the same safety property.

An operation with no finite cost bound MUST be given one, stopped
when the remainder is consumed, or refused. When a reserved
operation completes, the resource atomically commits the actual
debit — never more than it reserved — and releases the rest.

After a failure the resource MUST restore its ledger without rolling
back committed debits, and MUST preserve or reconcile any
reservation of unknown outcome before admitting work that could
exceed the cap.

A resource SHOULD report each debit in its response and MUST debit
exactly once per chargeable operation it serves. Retry, idempotency,
partial-response, and rounding semantics are API surface, pinned by
profiles; none of them may violate the accounting invariant.

# Budget State {#state}

A resource that enforces budgets MUST publish a `budget_endpoint`
member in its AAuth resource metadata document. Its value is an
absolute HTTPS URL with no query or fragment, on the same origin as
the resource identifier. This same-origin rule prevents an agent from
sending its auth token to a different party based on untrusted
metadata.

The agent makes a signed `GET` request with its auth token. The
resource verifies the token, signature, `mission`, and `budget` claims
before looking up or returning state. The token's `mission` claim
selects the meter; if the request also carries `AAuth-Mission`, it
must match as specified in {{claim}}. A successful response has
status 200, content type `application/json`, and the following
members:

~~~ json
{
  "active": true,
  "budget": { "amount": "2.00", "currency": "USD" },
  "spent": { "amount": "0.31", "currency": "USD" }
}
~~~

`active`:
: REQUIRED. A boolean. Whether the resource still serves budgeted
  requests under this mission: `false` once the cap is reached or
  the resource has otherwise stopped serving the grant.

`budget`:
: REQUIRED. The granted cap, as committed: an object with `amount`
  and `currency` as defined in {{budgets}}.

`spent`:
: REQUIRED. Cumulative debits to date, in the same shape. Remaining
  budget is the difference. Unlike a granted `amount`, a `spent`
  amount can be zero.

The response:

- MUST be an internally consistent snapshot including every debit
  committed before it, with `spent` excluding outstanding
  reservations and never exceeding `budget`;
- MUST carry `Cache-Control: no-store`; and
- MUST NOT include `sub` or any other identity claim.

The endpoint is advisory: concurrent work can consume the reported
remainder immediately after the snapshot. Agents SHOULD read it
before starting work whose cost is significant relative to the
remainder.

# Exhaustion {#exhaustion}

When no further chargeable work for a request can be admitted while
preserving the accounting invariant, the resource refuses that
request with HTTP status 402 and the error code `budget_exhausted`:

~~~ json
{
  "error": "budget_exhausted",
  "error_description":
    "The mission's remaining budget cannot cover this request."
}
~~~

The body shown is AAuth's flat error shape; a profile MAY pin its
API's native error carriage instead. The code means the remaining
budget is insufficient for this request; the refusal MUST NOT itself
create a debit. An agent MAY retry with a smaller finite cost bound
when the API permits; increasing the cap requires a new mission.

Payment failures outside the cap (an empty prepaid balance, a failed
settlement) belong to the resource's payment relationship with the
person. They are signaled with status 402 per AAuth and named by the
profile, not by this document.

# Conformance {#conformance}

An implementation conforms in one of four roles.

A **budgeted resource**:

- publishes `budget_endpoint` in its resource metadata and serves it
  per {{state}};
- rejects budgeted access on tokens lacking the `mission` or
  `budget` claims, or carrying values it cannot enforce, failing
  closed ({{claim}});
- preserves the accounting invariant under concurrency and recovery,
  meters per {{metering}}, and signals exhaustion per {{exhaustion}};
  and
- keeps identity out of budget-state responses.

A **budgeted agent**:

- proposes caps via `budgets` and reads the granted values from the
  approved blob ({{consent}});
- verifies that the approved blob carries a `budgets` member before
  treating the grant as budgeted: a PS that does not implement this
  extension may approve the mission without one;
- carries the mission reference in `AAuth-Mission` when AAuth
  requires it and verifies that issued auth tokens carry the expected
  `mission` and `budget` claims; and
- treats `budget_exhausted` as a refusal, retrying only with a smaller
  bounded operation or requesting a new mission decided by the person
  ({{immutability}}).

A **Person Server**:

- renders consent per {{consent}} and supports granting lower
  amounts;
- returns the granted entries in the approved blob;
- issues a token carrying a `budget` claim only while the mission is
  active and only where the token audience equals the entry's
  `resource` ({{claim}}); and
- carries the `budget` claim in auth tokens it issues, or conveys the
  granted entry using the `budget` federation parameter in {{claim}}.

A **budget-aware Access Server**:

- accepts `budget` only from an authenticated Person Server in the
  PS-to-AS token request;
- carries the resource token's mission reference into the auth token
  and verifies the exact equality among the resource token's issuer,
  `budget.resource`, and the auth-token audience; and
- copies the complete `budget` value into the auth token or refuses
  issuance, as specified in {{claim}}.

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
reference and budget are issuer-signed in the auth token used to
verify each request: what the person approved is what every call is
bound to, byte-exact. A PS that cannot render a proposed entry rejects
the proposal rather than approving what the person could not see
({{budgets}}).

**Accounting state is authorization state.** Atomic reservation,
commit, and durable recovery are necessary to the hard-cap claim. A
resource that rolls its meter back, races independent workers, or
charges after service without first bounding the charge can exceed
the approved amount. Such an implementation is not conformant even
if each worker is locally correct.

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

IANA is requested to register the following claim in the "JSON Web
Token Claims" registry established by {{RFC7519}}:

| Claim Name | Claim Description | Change Controller | Reference |
|---|---|---|---|
| `budget` | Mission budget for the token audience | IETF | {{claim}} |

The `budgets` mission member, `budget` federation parameter,
`budget_endpoint` resource metadata member, and `budget_exhausted`
application error code are defined on AAuth or application surfaces
that currently have no corresponding IANA registries.

--- back

# Complete Protocol Exchange {#e2e}

This appendix is non-normative. It walks the digest agent of
{{example}} through one full grant: proposal, approval, challenge,
issuance, spend, state, exhaustion, and completion. JWTs are
abbreviated, signatures are elided, and AAuth's interaction
machinery is reduced to its shape; AAuth's own examples govern the
substrate details. Later messages also elide `Signature-Input` and
`Signature`; every request remains signed as the first ones show.

The agent `aauth:scout@agents.example` proposes a mission at the
person's PS, asking for a $5.00 cap:

~~~ http-message
POST /mission HTTP/1.1
Host: ps.example
Content-Type: application/json
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key");created=1784365200
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "description":
    "Compile a daily digest of Doppler launch coverage this week.",
  "tools": [
    { "name": "search.query",
      "description": "Web search at api.search.example" }
  ],
  "budgets": [
    { "resource": "https://api.search.example",
      "amount": "5.00",
      "currency": "USD" }
  ]
}
~~~

Review is asynchronous; the PS defers while the person looks:

~~~ http-message
HTTP/1.1 202 Accepted
Location: https://ps.example/pending/m7Qk
Retry-After: 5
Content-Type: application/json

{ "status": "pending" }
~~~

The person approves at a lower cap, $2.00. Polling the pending URL
now returns the approved mission: the blob of {{example}}, byte for
byte, with its reference in the response header:

~~~ http-message
HTTP/1.1 200 OK
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
Content-Type: application/json

{
  "approver": "https://ps.example",
  "agent": "aauth:scout@agents.example",
  "approved_at": "2026-07-18T09:05:42Z",
  ...
  "budgets": [
    { "resource": "https://api.search.example",
      "amount": "2.00",
      "currency": "USD" }
  ]
}
~~~

The agent reads the granted cap from the blob: it asked for $5.00
and holds $2.00. It stores the bytes exactly as received and turns
to the resource. The first request is signed with its agent token
and carries the mission reference; the resource answers with the
challenge and a resource token:

~~~ http-message
GET /search?q=doppler+launch HTTP/1.1
Host: api.search.example
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
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

The resource token names the agent and copies the mission reference
from the header. Its payload:

~~~ json
{
  "iss": "https://api.search.example",
  "dwk": "aauth-resource.json",
  "aud": "https://ps.example",
  "agent": "aauth:scout@agents.example",
  "agent_jkt": "kV9CqXP3...",
  "scope": "search",
  "mission": {
    "approver": "https://ps.example",
    "s256": "q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
  },
  "jti": "rt_3Fq9xWv2",
  "iat": 1784386740,
  "exp": 1784387040
}
~~~

The agent exchanges it at the PS. The mission is active and the
resource token's issuer equals the entry's `resource`, so the PS
issues the auth token of {{claim}}, `budget` claim and all:

~~~ http-message
POST /token HTTP/1.1
Host: ps.example
Content-Type: application/json
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
Signature-Input: sig=("@method" "@authority" "@path"
    "signature-key" "aauth-mission");created=1784386790
Signature: sig=:...:
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "resource_token": "eyJ..resource-token..",
  "justification": "Search for today's launch coverage."
}
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{ "auth_token": "eyJ..auth-token..", "expires_in": 3600 }
~~~

The agent retries the search, now signed with the key the auth
token's `cnf.jwk` names. The resource verifies token, signature,
and reference equality, serves, and debits the grant, reporting the
debit in its own carriage (illustrative here; profiles pin it):

~~~ http-message
GET /search?q=doppler+launch HTTP/1.1
Host: api.search.example
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
Signature-Key: sig=jwt;jwt="eyJ..auth-token.."
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json

{
  "results": [ "..." ],
  "debit": { "amount": "0.01", "currency": "USD" }
}
~~~

Three days in, each hour's auth token re-obtained through the same
exchange, the agent checks the meter before a deeper crawl, at the
endpoint the resource metadata names in `budget_endpoint`:

~~~ http-message
GET /budget HTTP/1.1
Host: api.search.example
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
Signature-Key: sig=jwt;jwt="eyJ..auth-token.."
~~~

~~~ http-message
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "active": true,
  "budget": { "amount": "2.00", "currency": "USD" },
  "spent": { "amount": "0.31", "currency": "USD" }
}
~~~

The digest keeps running. When the remaining budget can no longer
cover a request, the resource fails closed:

~~~ http-message
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "error": "budget_exhausted",
  "error_description":
    "The mission's remaining budget cannot cover this request."
}
~~~

The week is over and the task is done, so instead of proposing a
top-off mission the agent proposes completion, spend in the summary:

~~~ http-message
POST /interaction HTTP/1.1
Host: ps.example
Content-Type: application/json
AAuth-Mission: approver="https://ps.example";
    s256="q2H2TTOaLlx16UWrc-h6TeSJFs31pfshIB_CqW0Rpl0"
Signature-Key: sig=jwt;jwt="eyJ..agent-token.."

{
  "type": "completion",
  "summary":
    "Week's digest complete. Spent $2.00 of the $2.00 budget."
}
~~~

The person accepts; the PS terminates the mission. Any later token
request under the reference fails closed:

~~~ json
{
  "error": "mission_terminated",
  "error_description": "This mission has ended."
}
~~~

Every message above rides AAuth surfaces unchanged. The extension's
whole footprint is the `budgets` member proposed and granted, the
`budget` claim issued, the meter and its endpoint at the resource,
and the 402 that ends the spending.

# Acknowledgments
{:numbered="false"}

This extension generalizes the budget mechanism of the Token Pony
Express {{TPX}}, which demonstrated the budget-as-grant pattern as an
OAuth 2.0 profile, onto AAuth's mission substrate. TPX-A
({{I-D.draft-mcguinness-aauth-budget-tpx}}) profiles this document
back onto that use case.
