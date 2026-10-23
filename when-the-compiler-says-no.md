# When the Compiler Says No

Applies to Punch 0.1.

Punch moves as many mistakes as it can to compile time. An operation without a policy, a failure without a status, a secret serialized to JSON, a path built with the wrong parameter: none of them compiles. That makes the compiler the first reviewer of every change, for you and for a coding agent. This guide explains how to read what it says, the messages you will meet most, and the fixes that keep the guarantee instead of working around it. Every message Punch gives is listed in the [compile errors table](../reference/compile-errors.md), and every lint rule in the [lint rules table](../reference/lint-rules.md).

After reading this guide, you will know:

- how to find the line that matters in a Crystal compile error;
- what Punch's messages tell you, and where they send you;
- the fix for the messages you will meet most often;
- why the quick fix is often the wrong one;
- how `punch check` reports what the compiler cannot see.

Before you start: nothing beyond a first app. The examples are the smallest programs that give each message, from `spec/fail/`.

## Reading a compile error

Many of Punch's checks run in macros, so a compile error often starts with the expansion that raised it: a few lines of generated code under `Which expanded to:`. Skip them. The message that matters is the last line that starts with `Error:`:

```text illustrative="the shape of a compile error from a Punch check"
In app/billing/invoices/operations/send.cr:12:3

 12 | struct Send < Punch::Command
      ^
Which expanded to:
...

Error: Billing::Invoices::Send needs a policy, @[Punch::Authorize(Policy)], or @[Punch::Public]: an operation without either would run for anyone (lib/punch/docs/authorization.md).
```

Punch's messages follow one pattern: what is wrong, named by your type; why, in a few words; the fix; and the page that explains it, as it is found in your app, under `lib/punch/docs/`. The location is in your own file, not in Punch's sources: an operation's mistake is reported at its `call` or its annotation.

`punch typecheck` checks the whole app in a few seconds without building it, after regenerating queries, translations, and requires. `punch typecheck --json` gives the errors as data, with file, line, column, and message, which is what an agent should read.

## The messages you will meet most

### No policy

```crystal fails=spec/fail/operation_without_policy.cr
module Notes
  struct Index < Punch::Query
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Reader) : Nil
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Notes::Index needs a policy, @[Punch::Authorize(Policy)], or @[Punch::Public]: an operation without either would run for anyone (lib/punch/docs/authorization.md).
```

Every operation says who may run it. Write the policy, or copy one from the namespace's `policies.cr`. `@[Punch::Public]` is the honest fix only when anyone on the internet may do this ([Authorization and Roles](authorization-and-roles.md)).

### No rate limit on an open Command

```crystal fails=spec/fail/command_without_rate_limit.cr
module Notes
  @[Punch::Public]
  struct Create < Punch::Command
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Nil
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Notes::Create is a Command that runs without a signed-in user, so it needs a rate limit, @[Punch::RateLimit(5, per: 1.minute, by: :ip)], or @[Punch::RateLimitExempt("reason")] (lib/punch/docs/rate-limits.md).
```

Add a limit. An exemption is honest only when something else bounds the Command, such as a provider's signature ([Rate Limits](rate-limits.md)).

### A failure without a status

```crystal fails=spec/fail/status_missing_in_union.cr
module Notes
  @[Punch::Status(200)]
  struct Added
    include JSON::Serializable

    def initialize
    end
  end

  # The failure forgot its status: it would answer 200 and commit.
  struct Missing
    include JSON::Serializable

    def initialize
    end
  end

  @[Punch::Public]
  @[Punch::RateLimitExempt("a fixture")]
  struct Create < Punch::Command
    struct Input
      include Punch::Input
    end

    def call(input : Input, db : Punch::Writer) : Added | Missing
      rand < 0.5 ? Added.new : Missing.new
    end
  end
end

Punch::Routing::Router.new(Punch::Database.memory)
```

```text
Error: Notes::Create#call can return Notes::Missing, which has no @[Punch::Status], in a result of 2 types of the app's: a type without one answers 200, so a failure would commit its writes and show as a success. Give every member a status: @[Punch::Status(200)] on the success, a status from 400 on for a failure, or `include Punch::Problem` in the failure (lib/punch/docs/operations.md, lib/punch/docs/openapi.md).
```

Include `Punch::Problem` in the failure type and give it its status ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)).

### A case that is not exhaustive

A new member of a result union, or a new state of a model, makes every `case … in` that does not handle it fail with Crystal's own message, `case is not exhaustive`, followed by the types it misses. That is the point of the union: add the branch. Never add an `else` that swallows what you did not think about.

### A secret leaving the process

```crystal fails=spec/fail/secret_to_json.cr
{token: Punch::Secret.new("sk_live_123")}.to_json
```

```text
Error: Punch::Secret cannot be serialized to JSON. Call `unwrap` explicitly if the value really must leave the process (lib/punch/docs/core-types.md).
```

Leave the secret out of the result. If it really must go somewhere, such as a header of a request to the provider, `unwrap` it at that line and nowhere earlier.

### A path with the wrong parameters

```crystal fails=spec/fail/path_missing_param.cr
module Invoices
  @[Punch::Public]
  struct Show < Punch::Query
    struct Input
      include Punch::Input
      @[Punch::PathParam]
      getter invoice_id : Int64
    end

    def call(input : Input, db : Punch::Reader) : Nil
    end
  end
end

Invoices::Show.path
```

```text
Error: Invoices::Show.path: missing path parameter `invoice_id` (Int64)
```

Pass every path parameter by name, with its type. A renamed parameter shows up here at compile time instead of as a 404 in production.

### A field type an input cannot parse

```crystal fails=spec/fail/input_union_type.cr
struct Amount
  include Punch::Input
  getter value : Int32 | String
end

Amount.from_json("{}")
```

```text
Error: Punch: Amount#value has type (Int32 | String), which Punch::Input cannot parse.
Use String, Bool, Int32, Int64, Float64, Time, an enum, a domain primitive (Punch::Primitive),
a nested Punch::Input, or an Array of one of these. Make it nilable (`T?`) to make it optional.
```

Choose one type for the field; for a value of several shapes, use optional fields and a `check` ([Input and Domain Types](input-and-domain-types.md#rules-across-fields)).

### A database row as a response

A result struct that holds a generated `Record` or `Row` fails with `… is a database row, not a response: return a result struct with the fields to show`. Build a result type with the fields the client should see, from the row, so the wire shape is declared and appears in `openapi.json`.

### A text parameter that is a pattern

`no overload matches 'Queries.customers_matching' with types Punch::Reader, q: String` means the parameter is a `Punch::LikeText`: pass `Punch::LikeText.new(text)`, which escapes the user's `%` and `_` ([Keeping SQL Safe and Fast](keeping-sql-safe-and-fast.md#searching-for-user-text)).

## The quick fix is often the wrong one

A check that fails is a guarantee that would break. The fix that makes the message go away is not always the fix that keeps the guarantee:

| Message | The quick fix | The honest fix |
|---|---|---|
| needs a policy | `@[Punch::Public]` | a policy |
| needs a rate limit | `@[Punch::RateLimitExempt("…")]` | a limit |
| has no `@[Punch::Status]` | `@[Punch::Status(200)]` on the failure | `include Punch::Problem` with its real status |
| case is not exhaustive | `else` | a branch per type |
| cannot be serialized | a `String` instead of a `Punch::Secret` | leave it out, or `unwrap` where it is used |
| `explain`: full scan | `-- scan: fine` | an index |
| a stale generated file | edit the file | change its source and regenerate |

When the honest fix is not clear, the page the message cites explains the rule and its reason.

## What `punch check` reports

Some conventions are not types, and `punch check` checks them after the build. Each failing step's detail says what to do; the full list is in the [steps table](../reference/check-steps.md):

- **Lint** (`lint`): Punch's own Ameba rules, such as SQL text built from a value, markup built from data, `ENV` read in app code, or a file in the wrong place, each with the path it belongs at. A rule is turned off for one line only, with a comment why ([Lint rules](../reference/lint-rules.md)).
- **Generated files** (`queries`, `i18n`, `requires`, `openapi`, `payloads`): a file that does not match its source. Run the command the message names; never edit the file.
- **The schema** (`schema`, `migrations`, `punchtables`): a schema file that does not match the migrations, a rebuild without its acknowledgement, a Punch table that an upgrade changed.
- **Query plans** (`explain`): a full scan or sort a query does not accept.
- **The built app** (`policies`, `limits`, `events`, `jobs`, `creds`): what the binary reports about itself.
- **Specs** (`specs`, `nocache`): the specs, then the specs again with the cache off.

`punch check --json` gives every step's status and detail as data. Every step must be `passed` or `skipped`.

## Common mistakes

**Reading the first line of the error.** It is often the macro expansion. Read the last `Error:` line.

**Fixing the message instead of the code.** The table above. A check that is loosened protects nothing.

**Turning off a lint rule for a file.** Disable it for one line, with a comment that says why, or not at all.

**Editing a generated file to make a check pass.** It is written again by the next run of its command. Change the source.

## Rules

- Read the last `Error:` line; follow the page it cites. ([Reading a compile error](#reading-a-compile-error))
- Fix the code so the guarantee holds; never silence a check with an exemption, an `else`, or a disabled rule that has no true reason. ([The quick fix is often the wrong one](#the-quick-fix-is-often-the-wrong-one))
- Run `punch typecheck` after every change, and `punch check` before finishing. ([Reading a compile error](#reading-a-compile-error), [What `punch check` reports](#what-punch-check-reports))

## Reference

- [Compile errors](../reference/compile-errors.md): every message, by topic, with its example.
- [Lint rules](../reference/lint-rules.md): every rule and what it reports.
- [`punch check` steps](../reference/check-steps.md): every step, what it does, and when it runs.
