+++
title = "Design Draft for `cargo-semver-checks` lint groups and CLI"
date = 2024-08-26

[taxonomies]
tags=["gsoc24", "rust"]

[extra]
issue = 6
+++

After implementing the core [manifest lint-level configuration](@/gsoc-update-1.md) for [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks), one of the next steps is adding some of the "nice to have"extra features that will make it much easier to use and configure `cargo-semver-checks` in projects' workflows.

Two of these features are **CLI config** and **lint groups**, and, because they are closely related, it can be helpful to think about how they will interact and design them in parallel (even though they might not be implemented at the same time).  Here are some of my considerations for what these features would look like in `cargo-semver-checks`.

## lint groups

A lint group, as the name might suggest, is a group of lints that share a certain property, to be configured all at once.  For context, `cargo-semver-checks` consists of many precise lints, such as `union_must_use_added` and `module_missing`, that each target one specific instance of a semantic versioning guarantee being broken.

Sometimes, though, it makes more sense to want to express something like "let's make all `#[must_use]` lints be warnings."  Currently, this can only be expressed with the following `lints` table:

```toml
[package.metadata.cargo-semver-checks.lints]
enum_must_use_added = "warn"
function_must_use_added = "warn"
inherent_method_must_use_added = "warn"
struct_must_use_added = "warn"
# ...
```

The granularity that `cargo-semver-checks` provides is super helpful, both for writing simple, testable lints and being able to control such precise behaviors, but sometimes, a project just wants to express something like

```toml
must_use_added = "warn"
```

This, unlike the current state, doesn't require the project to research and know about the internals of `cargo-semver-checks` and how it divides this semantic group into individual lints, and it is future-proof against additional lints of this category being added.  For example, the `union_must_use_added` lint was written around [two weeks ago](https://github.com/obi1kenobi/cargo-semver-checks/pull/845).

Thus, there are scenarios where it would be really helpful to be able to configure groups of similar lints.   

### how many groups?

One question that comes up is how many lint groups can a single lint belong to.  It could be one, so every lint group is disjoint from each other, or a lint could belong to zero-to-many lint groups.  The second option could be more expressive, as a lint could be part  of multiple semantic categories.  However, this would add a lot of complexity to the implementation.  Additionally, looking at prior art like [clippy](https://rust-lang.github.io/rust-clippy/stable/index.html#?groups=cargo,complexity,correctness,deprecated,nursery,pedantic,perf,restriction,style,suspicious), there is precedent for having one lint belong to a single lint group.  That way seems like the best path forward for `cargo-semver-checks`, but this can always be re-evaluated later on if we end up needing the addional expressivity of multiple lint groups per lint.

### two modes of configuration

As a reminder, each individual lint in `cargo-semver-checks` can be configured in two ways: the lint level (`allow`, `warn`, or `deny`), or the required semver bump when the lint finds breakage (`minor` or `major`).  Like lints, lint groups will be able to configured in these two ways, with the same syntax for specifying these as individual lints. 

### configuration precedence

What happens when a lint group is configured at the same time as a lint belonging to that group? How do they interact? In the cargo `[lints]` table, which `cargo-semver-checks` currently emulates, each configuration entry has a member `priority` (set to 0 if omitted).  Lower numbers (which can be negative) have lower priority, and if a configuration entry for a lint has a higher priority, it will override the lower priority.  Since `cargo-semver-checks` aims to be compatible with the `[cargo]` lint table, it makes sense to copy this behavior, although the edge cases (e.g., if a lint group containing a lint and that lint are configured at the same level) can become complex, so this will need testing for these behaviors.  Currently, there is a clippy warning `lint_groups_priority` for this specific case, so `cargo-semver-checks` could also produce warnings in configurations like these.  Note that the order of the fields in the `[lints]` table does not impact the lint sorting and precednece.

Additionally, as discussed below, priority will also have to be a consideration when specifying configuration through CLI flags.

### dynamic groups

In `rustc` linters, there are also dynamic groups, like `all` and `warnings` which allow the user to configure all defined lints or all lints currently configured to be warnings (e.g., to `deny` all warnings as hard errors).  These could be useful for expressiveness to quickly enable all lints/warning checks, but this does not seem like a crucial feature for the initial implementation.  Eventually, we could have dynamic lint groups for all currently-configured `major` and `minor`-level lints as well.

### grouping the lints

Here is a rough grouping of the currently-present lints in `cargo-semver-checks`.  This idea skews more specific (as opposed to e.g., clippy's `correctness`), but this part I would love feedback on.  Lists of lints by group in the appendix, linked for each group.

- [`must_use_added`](#must-use-added) - all the different instances of adding `#[must_use]`
- [`item_missing`](#item-missing) - when a public item is removed [^2]
- [`abi_changed`](#abi-changed) - when the `repr` or ABI of a public item changes [^3]
- [`non_constructible`](#non-constructible) - when a struct or enum variant is no longer constructible
- [`impl_removed`](#impl-removed) - when a struct/enum/union no longer implements a trait [^5]
- [`doc_hidden_added`](#doc-hidden-added) - when an item becomes `#[doc(hidden)]` and is removed from the public API
- [`safety_changed`](#safety-changed) - the safety of a function/item is changed in a way that is breaking
- [`guarantee_broken`](#guarantee-broken) - a sort of more general category of lint changes that can cause downstream crates to be broken (sometimes subtly) by removing guarantees of a previous API.  This is more general, and could potentially be broken up into smaller groups.

This would be helpful in situations above.  Clippy's lint groups are more general categories like `restriction`, `correctness`, etc., which could also be a direction to go in.  For example, there could be some warn-by-default lints that would fit nicely in a `suspicious` group.  This is where it could be helpful to have the additional expressiveness of having multiple lint groups for a single lint, but it seems possible to have a creative mix of more targeted/more general categories with only one group per lint, avoiding the additional complexity of many groups per lint.

## CLI config

Another feature is being able to configure lints and lint groups using command-line flags.  This is a more complicated feature, and we're [tracking user demand](https://github.com/obi1kenobi/cargo-semver-checks/issues/827) to see if it's desired to add.  It's helpful, though to reason about the interactions a potential command-line configuration interface would have with lint groups before either feature begins to be implementated.  

### prior art

`rustc`-based linting tools (such as clippy) have CLI syntax like:

```
--allow (-A) |
--warn (-W) |
--deny (-D) |
--forbid (-F) |
--force-warn 
<lint_or_group_name>
```

for example, `cargo clippy -- --deny clippy::undocumented-unsafe-blocks --allow unsafe-code`.

`cargo-semver-checks` uses these names for lint levels in the current Cargo.toml manifest configuration (but doesn't currently support `forbid` or `force-warn`, and it could help for interoperability in the Rust tooling ecosystem to use similar CLI flags.

### configuring version

However, `cargo-semver-checks` also allows another aspect of each lint to be configured: it's required semantic versioning bump when the lint occurs.  A CLI suite would need to expose this configuration avenue as well.  For example:

```sh
$ cargo semver-checks -- --major must_use_added --minor function_missing
```

It's tempting to add a shorthand for configuring lint level and required semver bump for the same lint/group at once, but this seems to me like a lot of additional complexity for no real gain in expressiveness.

Just like with manifest configuration, if a lint/group is configured for one aspect, the other aspect remains the default (or previously-configured value)

### precedence

When CLI and manifest both configure a lint or lint group, the CLI should take precedence.  This is also the case with e.g. clippy flags and the `[lints.clippy]` Cargo.toml table.  

Fortunately, command-line flags are provided in a deterministic sequence, so we don't need a `priority` key as we did in the manifest table: we just define (like other linters) the later flags taking precedence over the former flags, regardless of whether they are in lint groups or not.  For example:

```sh
$ cargo semver-checks -- --allow must_use_added --deny enum_must_use_added
$ cargo semver-checks -- --deny enum_must_use_added --allow must_use_added 
```

The first invocation would require a version bump if `#[must_use]` was added to a pub enum, because the `enum_must_use_added` flag took precedence over the lint group.  In the second example, however, it would be the opposite, because the blanket `allow` came after the targeted `deny` and thus takes precedence.

It's important to note that these precedence rules apply to each individual property of the lint (that is, version and lint level), meaning if we had

```sh
$ cargo semver-checks -- --major must_use_added --warn enum_must_use_added
```

the `enum_must_use_added` lint would be major-level and a warning (and other `must_use_added` lintsallow would be major-level and an error).

### a less-nuclear option

The problem that command-line flags solve is being able to configure lints/lint groups in an environment where the Cargo.toml manifest is not available to use for this purpose.  This could be in CI, using precomputed Rustdoc JSON files, etc., where `cargo-semver-checks` does not have access to the Cargo.toml manifest where lints would be configured.  

One alternative solution that avoids some of this complexity is adding a `--lint-config` flag that lets the user provide a path to a file that contains the lint configuration table.  With this, we only have to reason about precedence rules and edge cases in one format (the manifest configuration table), but there is a way to configure lints in environments where it's not currently possible.

Like command-line flags, any items in this table would take precedence over items in the package or workspace `Cargo.toml`.  One disadvantage of this over command-line flags is that it does require another file to be created, managed, checked into source control, etc., but it does solve the problem that CLI flags also do while using existing infrastructure.

## conclusion

I'd love to hear any feedback at this stage about lint groups, CLI config, or any of the problems, solutions, and implementation details proposed in this post.  You can find my sketch of the grouped current lints below in the appendix.

## appendix: existing lint groups

### `abi_changed` {#abi-changed}

- `enum_repr_int_changed`
- `enum_repr_int_removed`
- `enum_repr_transparent_removed`
- `exported_function_changed_abi`
- `function_abi_no_longer_unwind`
- `function_changed_abi`
- `function_export_name_changed`
- `repr_c_removed`
- `repr_packed_added`
- `repr_packed_removed`

### `non_constructible` {#non-constructible}

- `constructible_struct_adds_field`
- `constructible_struct_adds_private_field`
- `constructible_struct_changed_type`
- `enum_struct_variant_field_added`
- `enum_tuple_variant_field_added`
- `struct_marked_non_exhaustive`
- `struct_with_pub_fields_changed_type`
- `tuple_struct_to_plain_struct`
- `unit_struct_changed_kind`
- `variant_marked_non_exhaustive`

### `doc_hidden_added` {#doc-hidden-added}

- `enum_now_doc_hidden`
- `enum_struct_variant_field_now_doc_hidden`
- `enum_tuple_variant_field_now_doc_hidden`
- `function_now_doc_hidden`
- `inherent_associated_const_now_doc_hidden`
- `inherent_method_now_doc_hidden`
- `pub_module_level_const_now_doc_hidden`
- `pub_static_now_doc_hidden`
- `struct_now_doc_hidden`
- `struct_pub_field_now_doc_hidden`
- `trait_associated_const_now_doc_hidden`
- `trait_associated_type_now_doc_hidden`
- `trait_method_now_doc_hidden`
- `trait_now_doc_hidden`
- `union_now_doc_hidden`
- `union_pub_field_now_doc_hidden`

### `guarantee_broken` {#guarantee-broken}

- `enum_marked_non_exhaustive`
- `enum_variant_added`
- `function_const_removed`
- `function_parameter_count_changed`
- `method_parameter_count_changed`
- `pub_static_mut_now_immutable`
- `trait_newly_sealed`
- `trait_no_longer_object_safe`
- `type_marked_deprecated`


### `impl_removed` {#impl-removed}

- `auto_trait_impl_removed`
- `derive_trait_impl_removed`
- `sized_impl_removed`


### `item_missing` {#item-missing}

- `enum_missing`
- `enum_struct_variant_field_missing`
- `enum_tuple_variant_field_missing`
- `function_missing`
- `inherent_associated_pub_const_missing`
- `inherent_method_const_removed`
- `inherent_method_missing`
- `module_missing`
- `pub_module_level_const_missing`
- `pub_static_missing`
- `struct_missing`
- `struct_pub_field_missing`
- `struct_repr_transparent_removed`
- `trait_method_missing`
- `trait_missing`
- `trait_removed_associated_constant`
- `trait_removed_associated_type`
- `trait_removed_supertrait`
- `union_field_missing`
- `union_missing`
- `enum_variant_missing` missin

### `must_use_added` {#must-use-added}

- `enum_must_use_added`
- `function_must_use_added`
- `inherent_method_must_use_added`
- `struct_must_use_added`
- `trait_must_use_added`
- `union_must_use_added`

### `safety_changed` {#safety-changed}

- `function_unsafe_added`
- `inherent_method_unsafe_added`
- `trait_method_unsafe_added`
- `trait_method_unsafe_removed`
- `trait_unsafe_added`
- `trait_unsafe_removed`
