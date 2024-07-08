+++
title = "GSoC Update #1"
date = 2024-07-08

[taxonomies]
tags=["gsoc24", "rust"]

[extra]
issue = 5
+++


Hey everyone! It's been a little while... Here's some updates and reflections on my first few weeks [^1] working on [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks)
for Google Summer of Code 2024.  _(You can read my previous blog posts in the [`gsoc24` tag](/tags/gsoc24/) for an intro to what I'm doing.)_

## manifest configuration

A big feature of my project was to add lint-level configuration in the `[package.metadata]` table of the `Cargo.toml` manifest when running `cargo-semver-checks`, and this feature
has been merged into the main repo.  Here's an example of what you can do now:

_refresher: `cargo-semver-checks` runs checks (lints) between versions of a library crate to make sure a new release is following [semantic versioning guarantees](https://doc.rust-lang.org/cargo/reference/semver.html).
For example, the `function_missing` lint checks whether a `pub` function has been removed (or renamed) in the current version, which, by default, requires a new *major* version release._

Cargo.toml:

```toml
[package.metadata.cargo-semver-checks.lints]
function_missing = "warn"
module_missing = "minor"
enum_missing = { lint-level = "deny", required-update = "minor" }
trait_missing = "allow"
```

With this configuration, users can now configure the *required semver update* (`major` or `minor`) and *level of severity* (`deny`, `warn`, `allow`) for any given check.

Why might this be useful? Say a library has different policies than the semver guidelines for one check, for example changing the [ABI of a function](https://github.com/obi1kenobi/cargo-semver-checks/blob/main/src/lints/function_changed_abi.ron).  If the library only considers this a minor-level change, versus `cargo` guidelines' major, they couldn't integrate `cargo-semver-checks` (e.g., into CI) without much difficulty because this check would not have the right behavior for this specific library's needs.  In doing so, they miss out on checking the other [70+ checks](https://github.com/obi1kenobi/cargo-semver-checks/tree/main/src/lints) just because of this one discrepancy, which could easily lead to other semantic versioning violations because, as `cargo-semver-checks` shows, computers are very good at checking these sorts of things automatically.

Now, the library could simply add `function_changed_abi = "minor"` to their Cargo.toml in the `[package.metadata.cargo-semver-checks.lints]` table and be able to configure, run, and integrate `cargo-semver-checks` into their versioning workflows.  This helps prevent semantic versioning violations, which lets dependent crates and the whole Rust ecosystem more confidently use libraries and upgrade safely to get bug fixes and enhancements without breaking their existing code using the guarantees of semver.

Additionally, by adding warnings and lint levels, `cargo-semver-checks` can now run checks on code that don't necessarily *require* a new version bump.  With this, we can create new `warn`-by-default lints that check suspicious code: changes that are not necessarily semantic versioning violations, but suggest that there could be a (more complicated) error that `cargo-semver-checks` can't check with 100% certainty yet.

### workspace configuration

Additionally, I added the ability to configure the lints at the package and the workspace level.  By adding the similar `[workspace.metadata.cargo-semver-checks.lints]` table, libraries can configure defaults for `cargo-semver-checks` for all packages in a Cargo workspace, and then override in `[package.metadata]` for individual crates if need be.

## what's next?

Here are the things I'm going to be working on after this:

- **testing testing testing**: there are lots of edge cases and tricky situations that came up while adding these features, and I want to make sure that `cargo-semver-checks` has the expected behavior in these cases, so I will add even more unit/integration tests to our test suite.
- **documentation/migration guide**: with a big new feature like this, we need lots of documentation for both users of `cargo-semver-checks` as a Rust library and as a binary tool, so I'm going to write even more on how people can use the new configuration in their own projects and code.
- **clean up CLI output**: in adding warnings, there were definitely areas for incremental improvement in how `cargo-semver-checks` displays this information to the user, so I'm going to work on improving it and making it more consistent.
- **other features**:
 - *CLI flags*: the ability to pass `--warn function_missing`, `--minor module_missing`, and like configuration to the `cargo-semver-checks` binary instead of/in addition to configuration in the `Cargo.toml` manifest
 - *lint groups*: the ability to configure multiple related lints at once (like in tools like `clippy`), for example a `suspicious` group of warn-by-default lints.
 - more features? What do you think is important for this tool to have? I'd love to hear what you think, in this blog's GitHub issue or on the project's [Zulip stream](https://rust-lang.zulipchat.com/#narrow/stream/421156-gsoc/topic/Project.3A.20Adding.20lint.20configuration.20to.20cargo-semver-checks)


## reflections

I can't give enough thanks to my GSoC mentor Predrag Gruevski, who has given endless helpful and insightful feedback on all of my changes, thinking of areas and use-cases that are so important to consider.  Additionally, I've been working on my git hygiene, breaking up pull requests into as small as possible to be easier to review and make sure they are correct (sorry again for the first massive one, hehe).  Additionally, it's been so fun seeing the progress so far on the other GSoC projects and the support of the Rust community as a whole.  Happy coding!


[^1]: Although the coding period of GSoC has been on for more than a few weeks, my school ends later than typical, so we basically pushed back the start and end dates for me.
