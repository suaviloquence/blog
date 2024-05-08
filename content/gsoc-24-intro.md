+++
title = "GSoC '24 Intro"
date = 2024-05-07

[taxonomies]
tags=["gsoc24", "rust"]
+++

I'm super excited to be working on [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks) this summer! I'll be posting updates here under the [`gsoc24` tag](/tags/gsoc24) as I work on it this summer.  I love the philosophy of the tool and I think it's a great example of the ideas of the Rust language and ecosystem that let people make correct code more easily.

# semantic versioning and `cargo-semver-checks`: what am i working on?

*this is a little introduction to the problem that `cargo-semver-checks` tries to solve, and how it solves it.  if you're already familiar with the project, feel free to jump to the [planned work](#planned-work) section.*

I can tell my friends and family that I'm "adding lint-level configuration to `cargo-semver-checks`," but judging by the blank stares I get, that doesn't make it any clearer what I'm actually working on.  However, we all, even non-programmers, have some experience with the problems that semantic versioning and `cargo-semver-checks` try to solve:

Have you ever updated an app on your phone only to open it and find out that they, for whatever reason, removed a feature you used to use?  It probably broke your workflow, and there probably was little to no warning whenn you went to update the app that this would happen.  This is essentially the problem that semantic versioning works to solve.

With a piece of software that uses semantic versioning (shortened form: semver), its software version takes the form `x.y.z`, where `x` is the *major* version number, `y` is the *minor* version number, and `z` is the *patch* number.  Take an application `cat-petter` with version `1.5.2`. If the developers just fixed bugs or added minor improvements, they can just bump the *patch* version and release `1.5.3`, and users of this app will know that they can upgrade from 1.5.2 to 1.5.3 without any breaking changes.  However, let's say the developers add a feature to pet *all* pets in a way that *doesn't break* the original workflow of petting cats (here's a [rust analog: generalizing a function](https://doc.rust-lang.org/cargo/reference/semver.html#fn-generalize-compatible)), they should bump the *minor* version to show that this feature was added to version `1.6.0`.

It's important to note that this only applies if it doesn't make the original use-case in 1.5.2 stop working.  Say for example, that the developers wanted (for some reason) to make `cat-sitter` pet dogs instead of cats.  This new version of the app would break for someone using 1.5.2, so they should release version `2.0.0`, bumping the major version number, to show this.

Thus, you, the user of the app, can see the version change when you go to update the app.  If it's just a patch bump, the contract of semantic versioning assures you that you can update it without breaking your workflow.  If it's a minor version bump, you will know that some new features were added, but it *generally* will still work the way you used to use it.[^1]  If it's a major version bump, though, you know that there is the possibility to break your workflow, so you should be careful and research your decision to upgrade.

## enter `cargo-semver-checks`

There's one problem, though: as a developer, keeping track of all these changes is *hard*.  For Rust library crate developers, there's a [cargo guide](https://doc.rust-lang.org/cargo/reference/semver.html) on minor and major-breaking changes, but it's long, and it's non-exhaustive.  It's a lot to keep track of.

Do you know what is good at remembering and keeping track of things given well-defined rules? Computers are.  Just like the philosophy of the Rust language is to eliminate entire classes of programming errors by checking your programs at compile time, we can also check for semantic versioning violations when we are ready to release a new version of our library.

[`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks) is a tool that checks library crates for whether they have upheld the semantic versioning guarantees, or whether they need to make a minor or major version bump based on how the library code has changed between versions.  It does this by applying lints, one check for each different way that major or minor changes can occur, over the library's API surface, using [Trustfall](https://github.com/obi1kenobi/trustfall) and [`rustdoc`](https://doc.rust-lang.org/rustdoc/index.html).  
It's a great tool that greatly improves the developer experience, and, by reducing semver violations, builds more trust in the semver guarantees across the whole Rust ecosystem.  A library crate can add it to their continues integration tests, and efficiently track whether they have accidentally broken semver before they push the changes.

# planned work: how am i enhancing `cargo-semver-checks` this summer? {#planned-work}

However, there are lots of reasons, especially for large, mature library crates, why a project can't adhere completely to the way `cargo-semver-checks` interprets the semantic versioning guidelines.  For example, one project may not consider it a minor-level change just to [add a public item](https://doc.rust-lang.org/cargo/reference/semver.html#item-new).  Currently, this would prevent them from adding `cargo-semver-checks` to their CI pipeline without a lot of potentially-hacky workarounds.  We don't want this! All libraries should be able to use this tool without having to agree completely with its default interpretation of the semver spec.

## lint-level configuration

We want to be able to configure two things about each individual lint in `cargo-semver-checks`: whether to `allow`, `warn`, or `deny` (i.e., raise an error) when we detect it, and what semver level (`major` or `minor`) is it a breaking change for.  

Here's a general outline of what I'm planning on adding to the tool:

### CLI configuration

Like `clippy` and other linting tools, we want to be able to specify the error and semver levels while running the tool: for example `cargo semver-checks [...] -Afunction_export_name_hidden -Dtrait_method_now_hidden=minor -Wrepr_c_removed=minor` would make the `function_export_name_hidden` allowed/not error at all, and the `trait_method_now_hidden` lint now only a minor-version bump, and make the `repr_c_removed` both a warning and only a minor-breaking change, and all the different permutations of this.

### Cargo.toml config

There is a [`[lints]`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-lints-section) table in Cargo.toml, but it's not yet available to configure third-party tools like `cargo-semver-checks` with.  As of now, we can make a `[package.metadata]` section using the same syntax as the lints table, and then move over to that when it's stabilized for third-party tools.  When this is added, a user will be able to configure `cargo-semver-checks` in their Cargo.toml like this:

```toml
[package.metadata.cargo-semver-checks]
# set the semver level to minor for this lint
trait_method_now_hidden = "minor"
# set the semver level to minor and only emit a warning
# instead of an error on violation for this lint
repr_c_removed = { level = "warn", semver = "minor" }
# ignore all lints of this type
function_export_name_hidden = "allow"
```

This will also be added to `[workspace.metadata]` to configure at the workspace level as well.

Additionally, I'll focus on adding great and thorough tests and documentation to these additions to make `cargo-semver-checks` even more correct and easy to use and adopt.  I'll be posting updates at least every week under the [`gsoc24` tag](/tags/gsoc24) here to document my progress.

## more possibilities

This added configuration also opens up some more possibilities I'll work on if time allows during the summer or after if needed:

 - Add a group of `suspicious` lints that are not necessarily semver-breaking on their own, but are suspicious; they indicate that something is *probably* wrong but there is a possibility it does not break semver.  These will use the new configuration feature of being `warn` by default instead of `deny`, because they are warnings, not errors.
 - Allow CI targets like GitHub actions be configured directly in the action configuration for lint-level configuration, which will make it even easier and more flexible to use `cargo-semver-checks` in a CI pipeline
 - Configure whether to apply a lint on a specific module or even item-level basis, such as with attributes as tools like `clippy` do

 I'm super excited to be able to work on `cargo-semver-checks` this summer and beyond! Feel free to watch the [`gsoc24` tag](/tags/gsoc24) for updates on what I'm doing.  You can also find me on GitHub [@suaviloquence](https://github.com/suaviloquence) or by email (listed on my GitHub profile).


# footnotes

[^1]: note that this is not always the case: for example, from the cargo semver guide: *"Some changes are marked as “minor”, even though they carry the potential risk of breaking a build. This is for situations where the potential is extremely low, and the potentially breaking code is unlikely to be written in idiomatic Rust, or is specifically discouraged from use."*

