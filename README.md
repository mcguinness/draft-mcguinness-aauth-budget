<!-- regenerate: off (edited by hand; set to on to let i-d-template regenerate) -->

# Budgeted Missions for AAuth

This is the working area for the individual Internet-Draft, "Budgeted
Missions for AAuth" (AAuth-Budget).

AAuth missions express intent and tools, never quantity. This
extension defines the budget: a hard cap on cumulative monetary spend
at one resource, proposed by the agent, approved by the person at the
Person Server, committed under the mission's `s256`, carried in every
applicable auth token, and enforced by the resource that meters its
own service. A budget is a damage cap, not a payment instrument: a
compromised agent spends at most the remainder.

The extension adds four names on existing AAuth surfaces: one mission
member (`budgets`), one auth-token claim also used in PS-to-AS
federation (`budget`), one resource metadata member and endpoint
(`budget_endpoint`), and one error code (`budget_exhausted`). It
deliberately leaves the pricing unit, the debit-reporting carriage,
the API error shape, and the payment model to profiles.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-aauth-budget/#go.draft-mcguinness-aauth-budget.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-aauth-budget)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-aauth-budget)

## Profiles

"TPX-A: An AAuth Profile for Metered LLM Inference Grants"
(draft-mcguinness-aauth-budget-tpx) is the AAuth-native sibling of
[TPX v0.2](https://tokenpony.dev/spec/). It pins the credit unit, the
`models` restriction, the OpenAI-compatible API surface, and the
balance model onto this extension.

* [Editor's Copy](https://mcguinness.github.io/draft-mcguinness-aauth-budget/#go.draft-mcguinness-aauth-budget-tpx.html)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-mcguinness-aauth-budget-tpx)
* [Individual Draft](https://datatracker.ietf.org/doc/html/draft-mcguinness-aauth-budget-tpx)

## Building the Draft

Formatted text and HTML versions of the draft can be built using `make`:

```sh
$ make
```

This requires that you have the necessary software installed. See
[the instructions](https://github.com/martinthomson/i-d-template/blob/main/doc/SETUP.md).

## Contributing

Contributions are welcome via GitHub issues and pull requests.
