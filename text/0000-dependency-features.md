- Feature Name: dependency-features
- Start Date: 2018-04-11
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Make `cargo` pass additional `--cfg feature=` flag to `rustc` for each feature
in each direct dependency. Active features of dependencies could then be read by
writing `cfg(feature = "dependency-crate-name/feature-name")`.

# Motivation
[motivation]: #motivation

Some crates expose certain features or APIs only when particular features are
activated. Unfortunately, if you are building a dependent crate which needs to
modify its implementation based on how one of your dependencies is configured,
your only option currently is to add a new feature to your crate, and tell
consumers to turn on that feature whenever the other is enabled.

This RFC proposes adding a mechanism for reading the feature status of direct
dependencies of your current crate, so that you can perform the correct
conditional compliation without requiring additional user action.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

In addition to the features defined by your crate's `Cargo.toml`, your
dependencies features are also exposed to your crate. This is done in the
following ways:

 * When defining a dependency in your `Cargo.toml`, you can enable a set of its
   features by adding a `features` key to that dependency's descriptor:

```toml
[dependencies]
# The `dep_feature` feature is enabled for the dependency `dep`.
dep = { version = "1.0", features = ["dep_feature"] }
```

 * You can enable a feature in a direct dependency when a feature defined in
   your `Cargo.toml` is enabled by referencing the feature with the dependency's
   name as a prefix:

```toml
[features]
# The `dep_feature` feature is enabled for the dependency `dep`
# whenever the `my_feature` feature is enabled.
my_feature = [ "dep/dep_feature" ]
```

 * When using `cfg` to conditionally compile code, features defined in direct
   dependencies can be tested using the qualified name, as in the `[features]`
   section:

```rust
// This function is only compiled if the `dep_feature` for the
// dependency `dep` is enabled.
#[cfg(feature = "dep/dep_feature")]
fn f() { ... }
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The flags passed to `rustc` when building a crate would be modified. In addition
to enumerating the current crate's enabled features, passing a `--cfg
feature={feature_name}` flag for each, each enabled direct dependency would also
have its enabled features enumerated, passing a `--cfg
feature={dependency_name}/{feature_name}` flag to rustc for each.

The features of non-direct dependencies would not be enumerated or exposed to
the current crate, as they aren't necessarially part of the public API of a
crate.

This would have the effect of adding additional config flags to `rustc` for
`feature = "{dependency_name}/{feature_name}"` checks in `#[cfg(..)]` attributes
and the `cfg!(..)` macro. No changes to `rustc` itself would be necssary.

# Drawbacks
[drawbacks]: #drawbacks

A crate in your dependency graph could change its behaviour when a feature is
enabled in another one of your dependencies, potentially leading to hard to
debug build failures when certain features are enabled.

This proposal provides no mechanism for enabling other features or optional
dependencies when a feature in a dependency is enabled, which cuts off some
potential usecases.

# Rationale and alternatives
[alternatives]: #alternatives

The primary alternative is to not do this. This would mean that a crate could
not conditionally compile code based on the enabled features of its
dependencies. Crates will continue to re-export dependency features, in order to
enable these extra features.

There are also alternatives in syntax and explicitness:

## Dependency Feature Dependencies

The `Cargo.toml` file could be modified to allow directly specifying
dependencies of features from dependencies, e.g.

```toml
[features]
# Declare a local feature which turns on and off code in your crate.
my_feature = []
# Declare that the feature `dep_feature` in your dependency
# `dependency` should enable `my_feature` when it is enabled.
"dependency/dep_feature" = [ "my_feature" ]
```

This would ensure that all crates which use this feature explicitly opt into it,
and could make use of this feature more visible to `cargo` and other tooling,
which could help mitigate drawback #1.

This would also provide a mechanism for enabling other features or optional
dependencies based on whether or not a feature is enabled in your dependency.

The disadvantage of this option is that it is likely more complex to implement,
and less intuitive to use (the author expected the behaviour described in the
body text). It also requires additional boilerplate feature definitions.

## Dependency `cfg` Support

Rather than modifying `cargo`, the change could be made in `rustc`. `cfg` would
be expanded to support the `in_crate($crate, $condition)` syntax, which
evaluates the `cfg` condition `$condition` in the configuration context of the
dependency named by `$crate`. Here's a strawman syntax example:

```rust
extern crate dependency;

// This function will only be built if the `dep_feature` feature
// was enabled in the dependency `dependency`.
#[cfg(in_crate(dependency, feature = "dep_feature"))]
fn f() { ... }
```

This would be performed by having the `--cfg` flags passed to `rustc` saved in
the `rlib`'s metadata. `rustc` would load this metadata when requested by the
`crate(..)` syntax in order to evaluate the condition.

This syntax has the advantage of working for all `--cfg` flags, rather than just
the `feature=` flags generated by `cargo`. It could be taught fairly easily.

Unfortunately, this could not be used to control other `cargo` features or
enable optional dependencies, as it is happening at a level below `cargo`.

# Prior art
[prior-art]: #prior-art

I'm not aware of prior art in this area. This feature is inspired by the
dependency feature syntax used in the `[features]` section of `Cargo.toml`, and
tries to extend that idea to add a useful feature for libraries.

# Unresolved questions
[unresolved]: #unresolved-questions

 - Which mechanism is best for achieving the goal of conditionally compiling code
   based on which features are enabled in dependencies?

 - Is this a valuable feature? Do the improvements in user ergonomics exceed the
   downsides?
