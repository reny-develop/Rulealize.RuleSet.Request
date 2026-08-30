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

**`as` is not optional.** An alias defaults to the identifier and may not contain a `.`, so an
entry that leaves it out is refused — and the message names a key you did not write:

```
/uses[0]/as: must be a name without '.', which separates a held rule set from its input.
```

Write the identifier exactly as the package is published, too. nuget.org treats a package name
as one string however it is cased; the runtime does not, so `rulealize.ruleset.request` is a
different rule set and the document that arrives is refused when it compiles.

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

and two things it may do, which are the whole of what holding a rule set is.
[`example/roster.json`](example/roster.json) is a working document that does both.

**Refuse.** `held` is asked with the candidate in scope under the name this document gave its
parameter, so this is where a guard that spans both states is written:

```json
"held": {
  "req": {
    "raise": {
      "when": {
        "op": "logic.not",
        "value": {
          "op": "seq.any", "source": "$filled", "as": "f",
          "predicate": { "op": "cmp.eq", "left": "@f", "right": "@subject" }
        }
      }
    }
  }
}
```

*Do not raise a request for a shift already filled* — the guard neither half could write
alone, because the request half cannot see the roster and the roster half has never heard of
a request. **It may only refuse.** `held` is asked after this document's own guard, so nothing
written there grants a request that would not otherwise have been offered.

**Drive.** `fires` lets one of your inputs take a component's, so that granting the request
and acting on it are one decision rather than two states:

```json
"held":   { "req": { "grant": { "when": false } } },

"inputs": {
  "fill": {
    "fires":   [ { "held": "req", "input": "grant" } ],
    "effects": [ … your own write … ]
  }
}
```

`"when": false` hides `req.grant` so the only route to it is `fill`. The fired input still
goes through its own guard, so driving one is never a way past a rule this document wrote.

## Where it fits

Anywhere a thing may only happen once somebody said yes. The subject is a string and what it
means is the holder's business:

| The subject is | and the holder | |
| --- | --- | --- |
| a shift | fills it once the request is granted | [the example](example/roster.json) |
| an environment | deploys to it once the release is approved | |
| a discount | applies it once it is authorised | |
| a document | publishes it once review passed | |

**One request is one request.** A component holds one state, so holding this once gives one
request: several subjects to choose between, one asked for, and the case over when it is
settled. Two shapes for needing more:

- **`uses` may name it more than once**, under a different `as` each time, and each alias
  gets a state field of its own. Two independent grants over one subject is four-eyes
  approval written without a second document
- **an unbounded number is a case per request**, which is what a component being one state
  is telling you

**What it deliberately does not carry.** No reason on `deny` — that is presentation, and a
holder that needs one has its own state. No actor, no timestamps, no audit trail: those are
facts about a case that outlive the rule set that produced them, and the holder is what knows
where they go.

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

## Trying it

Needs [Rulealize.Cli](https://github.com/reny-develop/Rulealize.Cli):
`dotnet tool install -g Rulealize.Cli`.

**On its own.**

```sh
rulealize restore src/Rulealize.RuleSet.Request/ruleset/request.json
rulealize play    src/Rulealize.RuleSet.Request/ruleset/request.json --state state/example.json
```

```
Rulealize.RuleSet.Request@1.0.0 from 'state/example.json'
Choose by number. 'state' prints the position, 'q' stops.

    1. raise(subject: mon-am)
    2. raise(subject: mon-pm)
    3. raise(subject: tue-am)
>
```

Run it without `--state` and there is no legal input at all — the same document saying it has
not been told what may be requested.

**Held.** `--rulesets` says where the documents a composite holds are, so the example resolves
this one out of the package folder rather than a copy of its own:

```sh
rulealize restore example/roster.json --rulesets src/Rulealize.RuleSet.Request/ruleset
rulealize play    example/roster.json --rulesets src/Rulealize.RuleSet.Request/ruleset \
                  --state example/roster-state.json
```

```
    1. req.raise(subject: mon-am)
    2. req.raise(subject: mon-pm)
```

Take one, and `fill` is the only move left besides `req.deny` — `req.grant` is hidden, because
the roster drives it. Take `fill` and the request is granted and the shift filled in one
transition:

```sh
rulealize apply example/roster.json "req.raise(subject: mon-am)" \
  --rulesets src/Rulealize.RuleSet.Request/ruleset --state example/roster-state.json > r1.json
rulealize apply example/roster.json "fill" \
  --rulesets src/Rulealize.RuleSet.Request/ruleset --state r1.json
```

```
fill applied to 'r1.json' -- terminal (granted)
```

`apply` writes the state to standard output and everything else to standard error, so redirect
with `>` and not `2>&1`.

## Building the package

```sh
dotnet pack src/Rulealize.RuleSet.Request -c Release
```

A package with no `lib` folder, holding this one document under `ruleset/`.
[What each property in the project file is for](https://github.com/reny-develop/Rulealize.Registry/blob/main/doc/publish.md#a-rule-set).

## License

Apache-2.0. The document is the package's content and is covered by it, as is everything else
in this repository.
