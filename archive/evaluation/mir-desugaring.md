# MIR desugaring

[Recently](https://rust-lang.zulipchat.com/#narrow/stream/146212-t-compiler.2Fconst-eval/topic/complexity.20of.20constness.20in.20the.20type.20system)
I (Oli) have proposed to add a magic generic parameter on all `const fn foo`,
`impl const Foo for Bar` and `const trait Foo` declarations. This generic
parameter (called `constness` henceforth) is forwarded automatically to all
items used within the body of a const fn. The following code blocks demonstrates
the way I envision this magic generic parameter to be created (TLDR: similar to
desugarings).

### Examples

#### Trait declarations

```rust
const trait Foo {}
```

becomes

```rust
trait Foo<constness C> {}
```

#### Generic parameters

```rust
const fn foo<T: Bar + ~const Foo>() {}
```

becomes

```rust
fn foo<constness C, T: Bar + Foo<C>>() {}
```

#### Function bodies

```rust
const fn foo() {
    bar()
}
```

becomes

```rust
fn foo<constness C>() {
    bar::<C>()
}
```

#### Call sites outside of const contexts

```rust
fn main() {
    some_const_fn();
}
```

becomes

```rust
fn main() {
    some_const_fn::<constness::NotConst>();
}
```

#### Call sites in const contexts

```rust
const MOO: () = {
    some_const_fn();
}
```

becomes

```rust
const MOO: () = {
    some_const_fn::<constness::ConstRequired>();
}
```

### Implementation side:

We add a fourth kind of generic parameter: `constness`. All `const trait Foo`
implicitly get that parameter. In rustc we remove the `constness` field from
`TraitPredicate` and instead rely on generic parameter substitutions to replace
constness parameters. For now such a generic parameter can either be
`Constness::Required`, `Constness::Not` or `Constness::Param`, where only the
latter is replaced during substitutions, the other two variants are fixed.
Making this work as generic parameter substitution should allow us to re-use all
the existing logic for such substitutions instead of rolling them again. I am
aware of a significant amount of hand-waving happening here, most notably around
where the substitutions are coming from, but I'm hoping we can hash that out in
an explorative implementation
