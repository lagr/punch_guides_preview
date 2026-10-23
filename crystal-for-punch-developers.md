# Crystal for Punch Developers

Applies to Punch 0.1.

Punch apps are written in Crystal, a compiled language with a syntax close to Ruby's and a type system that checks the whole program before it runs. You do not need all of Crystal to write a Punch app. This guide teaches the part the other guides use, through Punch code, and points out where a Ruby habit leads astray. It is not a Crystal tutorial; the [Crystal reference](https://crystal-lang.org/reference/) is.

After reading this guide, you will know:

- how to read every construct in the guides' examples;
- why Punch's values are structs, and what that means;
- how unions and `case … in` replace exceptions for expected failures, and what the compiler says when a branch is missing;
- what annotations, generics, and macros do in a Punch app;
- the Ruby habits that do not carry over.

Before you start: nothing. If you know Crystal, skim the last section, then move on.

## It looks like Ruby, and it is compiled

```crystal illustrative="a few lines of Crystal, for the syntax"
def greeting(name : String, formal : Bool = false) : String
  formal ? "Good day, #{name}." : "Hi #{name}!"
end

["Ada", "Grace"].map { |name| greeting(name) }   # => ["Hi Ada!", "Hi Grace!"]
```

Blocks, string interpolation, `do … end`, and most of the standard library's method names are as in Ruby. The difference is the types. `name : String` says what a method takes and `: String` what it returns. The compiler infers types where they are not written and checks every call in the program before it runs: a method called with the wrong argument, or on a value that might be `nil`, does not compile.

Punch leans on this. Most of its conventions are checked by the compiler, so a mistake shows up as you build, with a message that names the fix ([When the Compiler Says No](when-the-compiler-says-no.md)).

## Structs and classes

Crystal has two kinds of type. A **class** is a reference, as in Ruby: two variables can point at one object, and a change through one is seen through the other. A **struct** is a value: assigning it copies it.

Almost everything in a Punch app is a struct: operations, inputs, results, components, events, jobs, policies. They hold data that does not change after they are made, so copying is cheap and nothing is shared between threads by accident. `record` declares a struct with its fields and a constructor in one line, as the reference app's invoice states do:

```crystal source=reference/app/billing/invoices/invoice.cr select=Billing::Invoices::Invoice::Draft,Billing::Invoices::Invoice::NoLines
# app/billing/invoices/invoice.cr
# Tier 2 (lib/punch/docs/models.md): an invoice has a lifecycle, and its state decides what
# may happen to it. Each state is its own immutable type; a transition is a
# method that returns the next state. Invoices::Store turns rows into these
# types and back. Nothing here touches the database.
module Billing::Invoices::Invoice
  # ...

  # Being written: can be revised, sent, or deleted.
  record Draft,
    id : Invoices::ID,
    customer_id : Customers::ID,
    currency : Punch::Currency,
    lines : Array(Line),
    created_at : Time do
    include Billed

    def status : Status
      Status::Draft
    end

    def revise(*, currency : Punch::Currency = @currency, lines : Array(Line) = @lines) : Draft
      copy_with(currency: currency, lines: lines)
    end

    # Issues the invoice under *number*. An invoice without lines cannot
    # be sent.
    def send(number : Invoices::Number, at : Time) : Sent | NoLines
      return NoLines.new if lines.empty?

      Sent.new(id, customer_id, currency, lines, created_at, number: number, sent_at: at, due_at: at + Invoices::PAYMENT_TERM)
    end
  end

  # ...

  # Why a draft could not be sent.
  record NoLines

  # ...
end
```

`revise` does not change the draft; it returns a new one with `copy_with`. A model's state changes by returning the next state, never by setting a field.

`getter name : Type` declares a field with a reader, `getter? urgent : Bool` one whose reader is `urgent?`. There is no `attr_accessor` with a setter on Punch's values.

## Nil is a type, and unions are values

In Crystal, `nil` is not something any value may be. A variable of type `String` always holds a string. A value that may be absent has a nilable type, `String?`, which is short for the union `String | Nil`, and the compiler makes you deal with the `nil` before you use it:

```crystal illustrative="nil has to be handled before the value is used"
customer = Table.find(db, id)          # Record | Nil
customer.name                          # does not compile: undefined method 'name' for Nil
return NotFound.new unless customer    # after this line, customer is a Record

customer.name                          # compiles
```

A union can hold any types, not just `Nil`. Punch uses unions for every outcome an operation can have. `Invoice::Draft#send` above returns `Sent | NoLines`: sending works, or the draft has no lines.

## `case … in`: every case handled

`case … in` branches on the types of a union, and the compiler checks that every type has a branch. The reference app's invoice page calls the `Show` Query and handles both of its outcomes:

```crystal source=reference/app/billing/invoices/pages/show_page.cr select=Billing::Invoices::ShowPage::Output,Billing::Invoices::ShowPage#call
# app/billing/invoices/pages/show_page.cr
module Billing::Invoices
  # ...

  # One invoice, with its customer's name from a second query. A draft
  # can be sent or deleted from here; a refusal (a draft without lines)
  # shows this page again with the reason. Agents read it too, as
  # markdown with the two forms as action blocks. Open in a browser, it
  # shows a change made elsewhere (sent, paid, revised, deleted) by
  # refreshing itself.
  @[Punch::AgentReadable]
  @[Punch::Authorize(ByRole)]
  @[Punch::Forms(Send, Destroy)]
  struct ShowPage < Punch::Page
    # ...

    alias Output = DetailView | NotFound

    def call(input : Input, db : Punch::Reader, actor : Invoicing::Actor, clock : Punch::Clock) : Output
      case detail = Show.new.call(input, db, actor, clock)
      in NotFound then detail
      in Detail
        customer = Customers::Table.find(db, detail.customer_id).try(&.display_name)
        DetailView.new(detail, customer, actor)
      end
    end

    # ...
  end
end
```

If `Show` gains a third outcome, this page stops compiling, with Crystal's message `case is not exhaustive` and the type it misses. That is how Punch handles expected failures: as values the compiler makes every caller handle, not as exceptions someone might forget to rescue ([Results, Errors, and Status Codes](results-errors-and-status-codes.md)).

`case … when`, as in Ruby, is still there for values, but it is not checked for completeness. Use `in` for unions and enums.

## Modules and `include`

A module groups code and names, as in Ruby. In a Punch app, a module is also a namespace and a directory: `Billing::Invoices` is `app/billing/invoices/` ([Application structure](../structure.md)).

`include` mixes a module into a type. Punch's own behaviours come this way: `include Punch::Input` makes a struct an input, `include Punch::Problem` a failure result, `include Punch::Job` a job. Including one often adds checks too, run by the compiler, such as the error for a job without `initialize` ([Background Jobs](background-jobs.md#running-on-a-schedule)).

`abstract def` in a module says what a type that includes it must define; the reference app's `Invoice::Billed` declares `abstract def lines`, and each state provides it.

## Generics

A type can take other types as parameters, like an `Array(String)`. In Punch you meet:

| Type | Means |
|---|---|
| `Punch::Grant(Send)` | proof that `Send`'s policy allowed the call |
| `Punch::Events(InvoiceSent \| InvoicePaid)` | publishing exactly these events |
| `Punch::Jobs(Remind)` | enqueuing exactly these jobs |
| `Punch::Cache(ID, Balances)` | a cache from customer IDs to balances |
| `Punch::Primitive(String)` | a domain primitive stored as a string |
| `Punch::Secret(String)` | a string that must not leak |

The parameter is what the compiler checks against: publishing an event that `Punch::Events(…)` does not list does not compile.

## Annotations

An annotation attaches information to a type or method that Punch reads at compile time:

```crystal illustrative="the annotations of the reference app's Send"
@[Punch::Authorize(ByRole)]
@[Punch::RateLimit(30, per: 1.hour, by: :actor)]
@[Punch::Idempotency(:required)]
struct Send < Punch::Command
```

They look like Ruby's DSL calls in a class body, and they are not code that runs: they are data the compiler checks. A policy that does not exist, or a rate limit with a key the input does not have, is a compile error.

## Constructors

`def initialize(@invoice_id)` both takes the argument and stores it in the field `@invoice_id`, whose type comes from the getter. A type with a `private def initialize` can only be made by its own methods: that is how a domain primitive makes sure every value went through `parse` ([Input and Domain Types](input-and-domain-types.md#writing-your-own)).

## Requires

Crystal compiles one program from the files it is told to require. A Punch app requires none by hand: `app/requires.gen.cr`, which `punch requires:generate` writes and `punch typecheck`, `dev`, and `check` keep current, requires every file under `app/` in the order of its kind. Adding a file is adding a file.

## Macros

Crystal macros write code at compile time. Punch's routes, OpenAPI document, input parsers, and most of its checks are macros. You use them constantly and almost never write one. When a macro's check fails, the error shows the expanded code first and the message last; read the last `Error:` line ([When the Compiler Says No](when-the-compiler-says-no.md#reading-a-compile-error)).

## Many threads, no shared state

A Punch app runs on several threads at once, and a fiber, Crystal's lightweight thread, can move between them. So there is no thread-local storage to put "the current user" in, and no global that requests may share. What a request needs comes in through `call`: the actor, the clock, the database handle. Values are structs, so passing them around copies them. State that must be shared, such as a cache, is Punch's, behind its own locks.

## Ruby habits that do not carry over

- **`rescue` for control flow.** An expected failure is a result type. Exceptions are for bugs, and a rescued one usually hides one.
- **Monkey-patching.** Reopening a standard type to add a method changes it for the whole program. Write a function, or a type of your own.
- **`method_missing` and dynamic DSLs.** Crystal knows every method at compile time; there is nothing to intercept. Punch's declarations are annotations and typed fields.
- **Mutable shared state.** A class variable that requests change is a race on several threads. Pass values, and keep shared state in Punch's stores.
- **`nil` as a default for everything.** Make a field nilable only when absence means something, and then handle it.
- **Strings for everything.** A `String` that should be an email address is checked nowhere. Use the narrowest type ([Input and Domain Types](input-and-domain-types.md)).

## Rules

- Make values structs, and change state by returning a new value. ([Structs and classes](#structs-and-classes))
- Handle unions with `case … in`, one branch per type, never an `else` that swallows the rest. ([`case … in`: every case handled](#case--in-every-case-handled))
- Return expected failures; raise only for bugs. ([Ruby habits that do not carry over](#ruby-habits-that-do-not-carry-over))
- Get what a request needs through `call`, never from a global or a class variable. ([Many threads, no shared state](#many-threads-no-shared-state))

## Reference

- [The Crystal reference](https://crystal-lang.org/reference/): the language.
- [Core types](../core-types.md#results-are-unions): results as unions.
