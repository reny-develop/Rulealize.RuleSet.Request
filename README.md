# Rulealize.RuleSet.Request

One request, raised over something and then granted or denied — a
[Rulealize](https://github.com/reny-develop/Rulealize) rule set written to be **held**.

| | |
| --- | --- |
| Rule set id | `Rulealize.RuleSet.Request` |
| Package | [`Rulealize.RuleSet.Request`](https://www.nuget.org/packages/Rulealize.RuleSet.Request) |
| Inputs | `raise` `grant` `deny` |
| Holds | nothing |
| Draws on | TypeSchema, State, Comparison, Logic, Binding |

```json
"uses": [
  { "ruleSet": "Rulealize.RuleSet.Request", "version": "^1.0", "as": "req" }
]
```

The identifier is the package identifier, so nothing has to look it up.

## What it is a request *for*

**Not in this document.** `subjects` is state, not schema: the rule set says a request is
raised over one of the things that may be requested, and which things those are arrives in
the state document.

```json
"req": {
  "ruleSet": "Rulealize.RuleSet.Request@1.0.0",
  "data": {
    "subjects": ["mon-am", "mon-pm", "tue-am"],
    "stage": "draft",
    "subject": null
  }
}
```

A roster's shifts, an order's line items and a deployment's environments are three state
documents and this one rule set.

That is also why `state.initial` names no subjects. A held rule set opens where it opens — a
composite cannot write its component's initial state — so a demonstration instance written
into this document would be forced on every document that ever holds it. An empty `subjects`
offers no `raise`, which is the right answer for a case nobody has said anything about yet.

## What a holder gets

| | |
| --- | --- |
| `rec.at($req, "stage")` | `draft`, `review`, `granted` or `denied` |
| `rec.at($req, "subject")` | what was requested, or `null` before `raise` |

and `held` is asked with the candidate in scope under the name this document gave it:

```json
"held": {
  "req": {
    "raise": {
      "when": {
        "op": "logic.not",
        "value": {
          "op": "seq.any", "source": "$assigned", "as": "a",
          "predicate": { "op": "cmp.eq", "left": "@a", "right": "@subject" }
        }
      }
    }
  }
}
```

That is the guard neither half could write alone: the request half cannot see what is already
assigned, and the assigning half has never heard of a request. **A composite may only
refuse** — `held` is asked after this document's own guard, so nothing written there grants a
request that would not otherwise have been offered.

## The three inputs

| | |
| --- | --- |
| `raise(subject)` | from `draft`. The domain is `subjects`, read out of the state. Records `subject` and moves to `review` |
| `grant` | from `review`. Ends at `granted` |
| `deny` | from `review`. Ends at `denied` |

**A request is one request.** `subject` is written once and never again; a second request is a
second case, not a second `raise`. And **`deny` carries no reason** — a reason is
presentation, this document is a gate, and a holder that needs reasons has its own state to
keep them in.

## Trying it on its own

```sh
rulealize restore src/Rulealize.RuleSet.Request/ruleset/request.json
rulealize moves  src/Rulealize.RuleSet.Request/ruleset/request.json --state state/example.json
```

```
Rulealize.RuleSet.Request@1.0.0 from 'state/example.json' (ongoing)
raise(subject: mon-am)
raise(subject: mon-pm)
raise(subject: tue-am)
3 legal inputs, 5 candidates evaluated
```

Run against `state.initial` instead and there is no legal input, which is the same document
saying it has not been told what may be requested.

## Building the package

```sh
dotnet pack src/Rulealize.RuleSet.Request -c Release
```

A package with no `lib` folder, holding this one document under `ruleset/`.
[What each property in the project file is for](https://github.com/reny-develop/Rulealize.Registry/blob/main/doc/publish.md#a-rule-set).

## License

Apache-2.0. The document is the package's content and is covered by it, as is everything else
in this repository.
