# A parametric AST in Rust

## 1. The problem

Abstract syntax trees are an ubiquitous data structure in language developement. However many times I ended up having to duplicate type definitions to fit the needs of different parts of the compiler. I recently started writing a proof assistant in Rust and quickly realized I needed different AST types for parsing, type checking, term unification, etc. This is known as the [expression problem](https://en.wikipedia.org/wiki/Expression_problem).

The (simplified) "base" AST looks like this:

```rust
enum Term {
    // variable
    Var(String),
    // function application
    App(Box<Term>, Box<Term>),
    // function definition
    Fun {
        argument: String,
        body: Box<Term>,
    },
}
```

But very quickly arises the need for an AST containing type information:

```rust
enum TypedTermValue {
    Var(String),
    App(Box<TypedTerm>, Box<TypedTerm>),
    Fun {
        argument: String,
        body: Box<TypedTerm>,
    },
}

struct TypedTerm {
    value: TypedTermValue,
    type_: Type,
}
```

Or even an AST specialized for unification, in which terms can be inference variables:

```
enum UnificationTerm {
    InferenceVar(usize),
    Var(String),
    App(Box<UnificationTerm>, Box<UnificationTerm>),
    Fun {
        argument: String,
        body: Box<UnificationTerm>,
    },
}
```

This duplication can become quite cumbersome when you change your AST, as you have to change many almost-equal definitions with only slight adjustments from the base AST. Notice how the main pain point comes from the recursive-ness of the type, which means we cannot just compose `Term` in another type to add more information, as that would only add data to the root.

## 2. Higher kinded types

A simple and elegant solution would be to use higher kinded types, which would allow us to specify the type of subterms. 

You can think of kinds as "types for types". Concrete types such as `String` have kind `type` but generic types can be thought of as functions taking types and returning types, for instance `Vec: type -> type` (`Vec<T>`) or `HashMap: (type, type) -> type` (`HashMap<K, V>`). This feature exists in Haskell but is (still) missing in Rust. Higher kinded types allow us to take generic types as generic type arguments.

We could then make `Term` generic over an argument `Rec` of kind `type -> type`.

```rust
enum Term<Rec> {
    Var(String),
    // notice how we specialize `Rec` with `Term<Rec>`
    App(Box<Rec<Term<Rec>>>, Box<Rec<Term<Rec>>>),
    Fun {
        argument: String,
        body: Box<Rec<Term<Rec>>>,
    },
}
```

Sadly, this is not currently possible in Rust. If we try to do this, the compiler will complain:

```
error[E0109]: type arguments are not allowed on type parameter `Rec`
 --> src/main.rs:3:17
  |
3 |     App(Box<Rec<Term<Rec>>>, Box<Rec<Term<Rec>>>),
  |             --- ^^^^^^^^^ type argument not allowed
  |             |
  |             not allowed on type parameter `Rec`
  |
note: type parameter `Rec` defined here
 --> src/main.rs:1:11
  |
1 | enum Term<Rec> {
  |           ^^^
```

## 3. Using associated types

Luckily, Rust still has a feature in its type system we can use: trait associated types. We define a trait that will act as a type family:

```rust
trait TermRec {
    type Rec;
}
```

We can then define our base AST like this, taking our original base AST and replacing recursive fields with `R::Rec`:

```rust
enum RawTerm<R: TermRec> {
    Var(String),
    App(Box<R::Rec>, Box<R::Rec>),
    Fun {
        argument: String,
        body: Box<R::Rec>,
    },
}
```

However now our `RawTerm` type is not directly usable, we have to specialize it. To do that we introduce a (zero sized) marker type, implement `TermRec` on it with the current `Rec` associated type:

```rust
// our marker type
struct Base;

impl TermRec for Base {
    type Rec = RawTerm<Base>;
}

type Term = RawTerm<Base>;
```

Let's break down what happens when we specialize `RawTerm` with `Base`. We can replace `R::Rec` with `RawTerm<Base>`:

```rust
enum RawTerm<Base> {
    Var(String),
    App(Box<RawTerm<Base>>, Box<RawTerm<Base>>),
    Fun {
        argument: String,
        body: Box<RawTerm<Base>>,
    },
}
```

But recall we have introduced a type alias, so we can use that to make the definition more readable:

```rust
enum Term {
    Var(String),
    App(Box<Term>, Box<Term>),
    Fun {
        argument: String,
        body: Box<Term>,
    },
}
```

That's exactly our first definition of `Term`! We can use this trick to define our typed AST, where we add an extra piece of data to every node of the tree:

```rust
struct Typed;

impl TermRec for Typed {
    type Rec = TypedTerm;
}

struct TypedTerm {
    value: RawTerm<Typed>,
    type_: Type,
}
```

Let's define an AST specialized for unification, our base AST with a variant added representing inference variables:

```rust
struct Unifying;

impl TermRec for Unifying {
    type Rec = UnificationTerm;
}

enum UnificationTerm {
    Var(usize),
    Const(RawTerm<Unifying>),
}
```

Our base AST is now defined only once and the difference between `Term`, `TypedTerm`, `UnificationTerm` is much clearer.

## 4. Interlude: trait impl assertions

`where` clauses can be used to assert that a given type implements a given trait by adding a `where` clause to a non-generic function.
This is known as [trivial constraints](https://github.com/rust-lang/rfcs/blob/master/text/2056-allow-trivial-where-clause-constraints.md).

For instance, this code compiles because each type implements its corresponding trait:

```rust
fn _test()
    where String: Clone,
          Vec<i32>: Default {}
```

However if you try putting an unsatisfied contraint:

```rust
fn _test()
    where String: Clone,
          Vec<i32>: Default,
          Box<i32>: Copy {}
```

The snippet will no longer compile: 

```
error[E0277]: the trait bound `Box<i32>: Copy` is not satisfied
 --> src/main.rs:4:11
  |
4 |           Box<i32>: Copy {}
  |           ^^^^^^^^^^^^^^ the trait `Copy` is not implemented for `Box<i32>`
  |
  = help: see issue #48214
```

We'll use this in the next part.

## 5. Downsides

You may notice I never put `#[derive(...)]` in any snippet before. This is not just to make the snippet simpler.
Let's try adding a `#[derive(Clone)]` attribute to `RawTerm<R>` and then assert `Term` implements `Clone` using our trick from earlier:

```rust
#[derive(Clone)]
enum RawTerm<R: TermRec> {
    ...
}

...

fn _test()
    where Term: Clone {}
```

But as it turns out, `derive` attributes currently do not mix well with associated types:

```rust
  --> src/main.rs:26:11
   |
26 |     where Term: Clone
   |           ^^^^^^^^^^^ the trait `Clone` is not implemented for `Base`
   |
note: required for `RawTerm<Base>` to implement `Clone`
  --> src/main.rs:7:10
   |
7  | #[derive(Clone)]
   |          ^^^^^ unsatisfied trait bound introduced in this `derive` macro
   = help: see issue #48214
help: consider annotating `Base` with `#[derive(Clone)]`
   |
17 + #[derive(Clone)]
18 | struct Base;
   |
```

The compiler wants us to implement `Clone` on the marker type, in that case `Base`.
This is because `#[derive(Clone)]` generates an `impl` that looks like this:

```rust
impl<R: TermRec + Clone> Clone for RawTerm<R> { ... }
```

However this is not correct. This is an unfortunate limitation of this method, 
`derive` macros don't work and we need to implement all traits manually.
To do that, we need to add a trait bound to the associated type:

```rust
trait TermRec {
    type Rec: Clone;
}
```

And then implement `Clone` manually:

```rust
impl<R: TermRec> Clone for RawTerm<R> {
    fn clone(&self) -> Self {
        match self {
            Self::Var(arg0) => Self::Var(arg0.clone()),
            Self::App(arg0, arg1) => Self::App(arg0.clone(), arg1.clone()),
            Self::Fun { argument, body } => Self::Fun { argument: argument.clone(), body: body.clone() },
        }
    }
}
```

This is a bit annoying since you will need to update "obvious" trait implementations when changing the AST definition.
However, you can do that pretty quickly thanks to rust-analyzer by just writing the header:

```rust
impl<R: TermRec> Clone for RawTerm<R> {
}
```

And then using the "implement missing members" code action in your IDE.

## 6. Conclusion

We now have a way to deduplicate AST definitions! Arguably, this approach to the expression problem introduces a bit of complexity
to your types, but I personally think the overall benefit of having very concise derived ASTs outweights this initial mental overhead.
