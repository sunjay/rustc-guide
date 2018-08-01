# Designing Chalk Rules

Your goal when designing rules for chalk is to make certain **queries** produce
the results you expect given a Rust program. For example, let's say you
define a predicate `Implemented(Type: Trait)` which means that the type `Type`
implements the trait `Trait`. You also have the following example program:

```rust
struct Foo { }
struct Foo2 { }
trait Bar { }

impl Bar for Foo { }
```

You would expect the query `Implemented(Foo: Bar)` to result in `Yes` because
of the final line in the program above. In contrast, you would expect
`Implemented(Foo2: Bar)` to be `No` because there is no impl block that adds
an implementation of `Bar` for `Foo2`.

So how do we take that Rust program and produce these results? This is the
interesting challenge of developing rules for chalk. You need to think about
which "facts" or "predicates" you can extract from the Rust program, then
determine how to phrase your queries so that those facts lead to the conclusion
that you want.

This page is meant to act as a collection of helpful tips and advice that that
should help you do that. If you think of something that would be a helpful
addition to this, please add it!

## Choosing Predicates

You are allowed to use or introduce predicates for any information you think the
Rust compiler has available or could derive easily. Here, "easily" means you
don't have to look at other declarations of other items (other structs, enums,
etc.). The property you extract must be a property of just that one item. If it
involves multiple declarations, it should be combined in chalk using a rule. For
example, consider the following code:

```rust
struct Foo<T> { }
trait Bar { }
trait Spam { }

impl<T> Bar for Foo<T> { }
impl<T: Spam> Bar for T { }
```

One way to lower this is into the following predicates:

```rust
forall<T> { Implemented(Foo<T>: Bar) }
forall<T> { Implemented(T: Bar) :- Implemented(T: Spam) }
```

Here, we've taken each impl item and produced a rule that describes which trait
was implemented for which type. More formally, the `Implemented(Type: Trait)`
predicate means "Type implements Trait". It is a good idea to write out what
each predicate means in English when you define it so that you can remember
the role of each term.

The Rust compiler also has information about which types and traits come from
which crates. You can model that information in chalk too:

```rust
// In Crate "foo"
pub struct Foo { }
```

You could add a lowering rule that turns this into:

```rust
InCrate(Foo, foo)
```

The way you choose to model these predicates makes a huge difference in the
rules you can write with them. For example, if you don't care which crate `Foo`
comes from and only want to know whether it is external to the current crate
being compiled or not, you can lower it into something like this instead:

```rust
IsExternal(Foo)
```

This has less information, but allows you to write queries that don't care about
which specific crate `Foo` comes from. Choose your predicates wisely so that you
avoid overcomplicating your queries.

## Conditionally Applying Rules: Implication

Another technique you can use is implication. You can lower something into a
query that looks something like this:

```rust
forall<T1, T2, ...> { Consequence :- Condition1, Condition2, ..., ConditionN }
```

Implication is incredibly useful for applying rules only under certain
conditions.

For example, consider modelling downstream types. Downstream types are types
from crates that use the current crate as a dependency.

One way we can model this is by adding a predicate called `DownstreamType(T)`
which in English means "T is a downstream type." When we're compiling a Rust
program, we examine only the current crate being compiled and its dependencies.
We don't have access to any crates that depend on the current type. That means
that by the definition of `DownstreamType`, we should not produce any
`DownstreamType` predicates for any of the structs that we go through. The
consequence of this is that by default, `DownstreamType(T)` is not provable for
any type `T`.

```rust
// In Crate "foo"
struct Foo { }

// In the Current Crate "bar"
extern crate foo;
struct Bar { }
```

Using only `DownstreamType` as we've defined it, this would get lowered into:

```rust
// Nothing...
```

That's right. Nothing is produced because `DownstreamType(T)` isn't added for
anything when we lower the types in our program.

Even though we aren't lowering anything for the structs in our program, we can
add a rule for each trait that deals with downstream types. For example, let's
say that we wanted to model that downstream crates can implement traits for
their downstream types. We'll stick to just single type parameter traits and
leave out other cases. We can model this using the following rule produced for
every trait `MyTrait`:

```rust
forall<T> { Implemented(T: MyTrait) :- DownstreamType(T) }
```

In English, this says "For any type T, T implements MyTrait if T is a downstream
type."

> Note: This rule is actually a bit too strong to model this situation
> correctly. In the next section, we'll learn how to fix this problem.

By lowering each trait into this rule, we can now potentially prove the
`Implemented` predicate using the `DownstreamType` predicate. The problem is, as
we discussed, `DownstreamType` isn't provable for any type yet. In order to use
it, we need to introduce it somehow. We can do that using **implication**.

Whenever we want to prove something involving downstream types, we now need to
introduce `DownstreamType(T)` using implication.

```rust
exists<T> { Implemented(T: MyTrait) :- DownstreamType(T) }
```

In English, this query asks "Is there a type T where if T is a downstream type,
it implements MyTrait."

Let's say we have this program:

```rust
struct Foo { }
trait MyTrait { }

impl MyTrait for Foo { }
```

Using just what we've discussed so far this would lower into:

```rust
forall<T> { Implemented(T: MyTrait) :- DownstreamType(T) }
Implemented(Foo: MyTrait)
```

Now we can prove our query easily using the first rule in the lowered program.

If we were to query without introducing `DownstreamType` via implication, we get
a substitution back with the type `Foo`.

```rust
exists<T> { Implemented(T: MyTrait) }
```

This would unify with the second line in the lowered program and result in
`T = Foo`.

Our `DownstreamType` predicate has allowed us to model that downstream crates
can implement traits for downstream types. We used implication to introduce
a new downstream type and looked at how that can change the results of a query.

In reality, providing a definitive answer about downstream types is probably too
strong of an assumption. It would be nice if we could provide a more ambiguous
response such as "Maybe". Chalk provides a special goal that gives us the power
to do that. The "Maybe" response in chalk is called "Ambiguous" and it is
discussed in detail in the next section.

## Producing Ambiguous Answers

This section will build on the example from the previous section and show how we
can model the situation where a downstream crate *might* implement a trait for a
type using its own downstream types.

A special, internal-use-only goal `CannotProve` is designed to be used to
indicate when something cannot be proven true or false definitively.
`CannotProve` produces "Ambiguous" and is useful for when we want queries to
produce neither "Yes" nor "No" in certain situations.

When we discussed the prior example, we noticed that this rule was a bit too
strong:

```rust
forall<T> { Implemented(T: MyTrait) :- DownstreamType(T) }
```

In English, this rule says "Each type `T` implements `MyTrait` if `T` is a
downstream type". This is too strong because while we know that downstream types
*can* implement traits from crates they depend on, they are not *guaranteed* to
do that. That means that we cannot conclude definitively either way.

TODO: Finish this section


- [ ] It is okay to generate rules for every possible case (within reason)
  - [ ] Example: `LocalImplAllowed`
- [ ] Phrase things positively because negative reasoning can sometimes change things (i.e. IsExternal vs not { IsLocal })
  - [ ] Why negation as failure causes problems
- [ ] behavior of `Yes`, `No`, and `Ambig` in the context of `And`, `Or`, and `Not`
