+++
title = "How do Rust tools handle lint configuration?"
date = 2024-05-27

[taxonomies]
tags=["gsoc24", "rust"]

[extra]
issue="3"
+++

As I start to work on [adding more lint configuration](@/gsoc-24-intro.md) to [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks), I've been exploring the different ways that tools like `rustc`, `clippy`, and `cargo` handle linting and configuring the levels of each of their lints to make it more consistent for `cargo-semver-checks`'s configuration.
Some of it is internal compiler code that we can't really reuse, but there are crates like [`annotate-snippets`](https://crates.io/crates/annotate_snippets) are being developed to make it easier and more consistent to create linting tools (in this case, by providing an interface to render diagnostics).
We can also copy/take inspiration from the interfaces (e.g., command-line flags) of rustc and clippy to make configuration more consistent.

# lint levels

The biggest part of configuring lints is to specify how big a problem it is when this lint occurs. If we should raise an error, that's `deny`. If it should just be a warning: aptly `warn`. And if, through our configuration or the default configuration, it should not error or raise a warning when this lint is triggered, we `allow` that lint.

## a secret fourth option

There's another option in `rustc`, though, that is even stricter. If we `forbid` a lint, it's like a stricter version `deny`ing it. There are a [lot of ways](#configuration) to configure lints tools like `rustc` and `clippy`, and sometimes you `deny` a lint (such as the `unsafe-code` lint) at the module level, for example, but you can still `allow` or `warn` an individual `unsafe` block. If we want to prevent this from happening, we can `#![forbid(unsafe_code)]`, and this lint will now _always_ be an error, regardless[^forbid] of other configuration that would otherwise override a `deny` level.

## a fifth one?

This one was actually new to me when I was reading the [rustc lints page](https://doc.rust-lang.org/rustc/lints/levels.html). Similar to `forbid`, there is a `force-warn` level that will always make a lint emit a warning, even if it is configured to be `allow` _or_ `deny` at a higher-precedence config. Unlike `forbid`, though, `force-warn` can't even be suppressed by `--cap-lints allow`. However, it can't be set by an attribute like the other ones, and I personally have not seen it used in the wild yet.

## looking at `cargo-semver-checks`

The first three are unambiguously necessary to add to `cargo-semver-checks` to me, as the whole point of adding this more granular configuration to be able to specify the level of each check.

I'd love feedback on `forbid`, though. It requires special handling (to be able to override later configuration on the same lint), and right now, the initial plan is to only add three places to configure the lint: workspace and package `Cargo.toml` as well as through CLI flags, so adding it might not be as necessary as in `rustc`, where every module and submodule and item and field can have their own configuration, so it's a lot more helpful to be able to override something at an outer level.

That being said, if we do add module-level lint configuration, it might be helpful to have `forbid`, and as a Rust/cargo tool, users might expect to be able to forbid a lint in `cargo-semver-checks` as in other tools.

If you have any thoughts on this, I'd love to hear what you think! Feel free to post on the blog [GitHub issue](https://github.com/suaviloquence/blog/issues/3) or in the [project Zulip stream](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Adding.20lint.20configuration.20to.20cargo-semver-checks).

As for `force-warn`, I personally would need some convincing to see why it would be useful to add to `cargo-semver-checks`, as I haven't even seen it used in `rustc` lints yet (that I know of).

# configuration

The point of these lint levels is to be able to set them, of course, and there are so many different ways to configure them in the ecosystem.

## cli flags

rustc (and clippy, which uses rustc's interface) let the user pass `--allow`/`-A`, `--warn`/`-W`, `--force-warn`, `--deny`/`-D`, `--forbid`/`-F` with the qualified lint name (e.g., `clippy::absurd_extreme_comparisons` or `dead_code` == `rust::dead_code`) to set the lint level at the scope of the compile target. Ideally, a user of `cargo-semver-checks` would be able to do this for our checks.

_note: for rustc-registered lints, these can also be passed to `RUSTFLAGS`_

## Cargo.toml tables

The [`[lints]`](https://rust-lang.github.io/rfcs/3389-manifest-lint.html) ([cargo book](https://doc.rust-lang.org/nightly/cargo/reference/manifest.html#the-lints-section)) table in the Cargo manifest was added to declare package or workspace-scope lint levels. Currently, this only works with `rust`, `clippy`, and `rustdoc` lints, so `cargo-semver-checks` would not be able to use it as of yet. However, we can simulate the syntax in a `[{package,workspace}.metadata]` subtable until it is stable for third-party tools to have entries in the lints table.

## module/item attributes

For rustc lints, you can add configuration like `#![allow(lint)]` outer attribute to a module as well as attributes like `#[warn(lint)]` on an inner item itself to apply that configuration to just the module/item. Currently, `cargo-semver-checks` has a less granular version of this by adding `#[doc(hidden)]` to an item to exempt it from semver guarantees, but adding this is a breaking change.

## lint groups

Lints in rustc and clippy are organized into groups/collections of lints that can all be configured at once (e.g., `#[warn(clippy::pedantic)]`). It's definitely a goal of `cargo-semver-checks` to implement this as well, for instance with a `suspicious` group of warn-by-default lints that are not necessarily breaking changes at all times, but usually indicate breakage of a semver guarantee somewhere.

## precedence

When there are multiple levels set for a given lint, we have to figure out which one takes priority, which can get complicated:

- later CLI flags take priority over earlier CLI flags (`--deny clippy::cast_sign_loss --allow clippy::pedantic --warn clippy::cast_possible_truncation`) would make `cast_possible_truncation` warn, but `cast_sign_loss` allow (both lints are in the `pedantic` group).
- the `[lints]` table has a `priority` key for each entry, such that lower is a lower priority, and higher higher (default is zero if not set). Because in TOML key order is not guaranteed, the order of the lints in the lint table should not be used for configuration, and users should set `priority` instead if there is a conflict.
- `rustc` uses a stack of [`LintSet`](https://github.com/rust-lang/rust/blob/f00b02e6bbe63707503f058fb87cc3e2b25991ac/compiler/rustc_lint/src/levels.rs#L75)s, based on how deep the scope is where the configuration is added (i.e., an item is on the top of the stack, its module is in the middle, and command line flags are at the bottom of the stack)
- CLI flags override the `[lints]` table
  <details>
  <summary>test for this</summary>

  Cargo.toml:

  ```toml
  # ...
  [lints]
  unused_variables = "deny"
  ```

  main.rs:

  ```rust
  fn main() {
      unsafe {};
  }
  ```

  `$ cargo clippy`:

  ```
  cargo clippy
      Checking tst v0.1.0 (/tmp/tst)
  error: unused variable: `a`
   --> src/main.rs:2:9
    |
  2 |     let a = 0;
    |         ^ help: if this is intentional, prefix it with an underscore: `_a`
    |
    = note: requested on the command line with `-D unused-variables`

  error: could not compile `tst` (bin "tst") due to 1 previous error
  ```

  `$ cargo clippy -- -Aunused_variables`

  ```
  Checking tst v0.1.0 (/tmp/tst)
  Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s
  ```

  </details>

- and of course, if `forbid` is found at any point, it overrides any other configuration

Looking at `cargo-semver-checks`, we would need to calculate this precedence ourselves, at least right now. However, it seems like we would just need to worry about CLI flags, package tables, and workspace tables, and potentially `forbid`, as you can't annotate an item or module. Additionally, we want to be able to configure the _required semver version bump_ as well as the lint level. There is the future possibility of configuring this (i.e., tool-specified configuration) in the `[lints]` table, such as `enum_missing = { level = "warn", semver = "minor" }`, but this functionality is not available in the `[lints]` table yet. However, we can add it to our own `[package.metadata]` table, then integrate with `[lints]` when we can. However, we would need our own way to configure this at the command line, as there's little precedent for providing arguments to lints in tools like clippy.

# consistency

## annotate-snippets

[`annotate-snippets`](https://crates.io/crates/annotate_snippets) is a crate that creates `rustc`-like formatted diagnostics for code. You can provide it with level, text, and attach other related diagnostics to the error message.

`cargo`'s linting tool is [currently using it](https://github.com/rust-lang/cargo/blob/95eeafa3ba513a630d32aecf2818734aeb06b540/src/cargo/util/lints.rs#L6), but tools like `rustc` and `clippy` are using [rustc-internal diagnostic rendering](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_errors/struct.DiagCtxt.html)

## compiler lints

Tools like `clippy` register with `rustc`'s linter using [`declare_tool_lint!`](https://doc.rust-lang.org/stable/nightly-rustc/rustc_lint_defs/macro.declare_tool_lint.html). `rustc` then handles configuration of lint levels and running the lints as necessary, and the tools define the behavior.

## cargo lints

When lints have to run over a crate (including a `Cargo.toml` manifest), it makes less sense to use `rustc`'s lint handling. cargo's linting is relatively new, and it seems to roll its own configuration, at least for now. (see [`struct Lint`](https://github.com/rust-lang/cargo/blob/master/src/cargo/util/lints.rs#L263) and [`level_priority`](https://github.com/rust-lang/cargo/blob/95eeafa3ba513a630d32aecf2818734aeb06b540/src/cargo/util/lints.rs#L386))

## consequences for `cargo-semver-checks`

The lints in `cargo-semver-checks` are much closer to cargo's than those of rustc and clippy[^clippy-cargo]. Because we need access to the whole API of a crate (and a comparison baseline version of that crate), it doesn't seem very feasible to use compiler lints.

What `cargo-semver-checks` does differently than other tools is that lints are created _declaratively_ with a [Trustfall](https://github.com/obi1kenobi/trustfall) query over the crate's API, instead of as a Rust constant. This means that once/if cargo exposes the functionality for third-party crates to register cargo lints, it may not be plug-and-play (especially if it expects something with a `'static` lifetime, as we need to parse them at runtime).

I'm not sure how well we can integrate with cargo lints right now, and it seems like I will have to write my own level precedence calculator. However, we definitely want to design to integrate with cargo in the future, and if you have any suggestions for how to do that, I'd love to hear them. Additionally, maintaining interface (like CLI) compatibility with tools like rustc is also a goal as much as we can, even if we don't use the same lint mechanisms under the hood.

# acknowledgements

Thanks to Ed Page, Scott Schafer, and Predrag Gruevski for great info and pointers, especially about `annotate-snippets` and cargo's linting in the [Zulip thread](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Adding.20lint.20configuration.20to.20cargo-semver-checks).

Again, I'd love to hear any feedback, either in the Zulip, as a comment on the GitHub issue for this post, or by email.

---

[^forbid]: In `rustc`, you can pass `--cap-lints [level]` and it will suppress all lints at a stricter level by capping them to the passed level. This makes even something with `forbid` an `allow` or `warn` if capped.

---

[^clippy-cargo]: Although `clippy` has cargo integration and lints parts of Cargo.toml as well; see [`cargo/mod.rs`](https://github.com/rust-lang/rust-clippy/blob/76eee82e79e736c4cef6ee9f755f58e752b9f58a/clippy_lints/src/cargo/mod.rs) (it makes a `Cargo` lint pass, and calls `cargo metadata` - we could technically do this for `cargo-semver-checks`, but we would need a lot of refactoring, and it would be a little hacky)
