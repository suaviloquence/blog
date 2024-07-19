+++
title = "cargo-semver-checks lint configuration guide"
date = 2024-07-18

[taxonomies]
tags=["gsoc24", "rust"]
+++

Recently, I've added functionality to `cargo-semver-checks` to control both the lint level and required semver update for each check.

## definitions

- **check/lint** - `cargo-semver-checks` runs many checks between versions of a crate's public API, which each check for one specific semver-breaking behavior.  For example, the `function_missing` lint is triggered when a function in the crate's public API is removed in a new version. 
- **lint level** - For each of these checks, it is now possible to configure the level of severity for when this check occurs.  `deny` means it is a hard error when the check finds breaking behavior, which will require a version bump to resolve.  `warn` still runs the check and reports findings, but it is only a warning if it triggers and does not require (but it does suggest) a version bump.  `allow` means that the breaking behavior in this check is allowed in the code, so the check doesn't need to be run.
- **required (semver) update** - what version bump this check should require (for `deny`-level) or suggest (for `warn`) when the check is triggered.  For example, `function_missing` is `major` by default, so if a public function is removed between versions, the version needs a major version bump (e.g., 1.2.3 to 2.0.0 or 0.5.2 to 0.6.0).  This can be configured to `major` or `minor` (1.2.3 to 1.3.0 or 0.5.2 to 0.5.3).


## configuring checks

To configure the level and/or required update for a check, first find its id.  This will be in `snake_case` and is reported on error/warning or found as the file name in the [lints folder](https://github.com/obi1kenobi/cargo-semver-checks/tree/main/src/lints).  Let's use `function_missing` as an example.

Then, determine what the new lint level and/or required version update should be.  For our example, let's set the level to `warn` and the required update to `minor`.

In the `Cargo.toml` manifest for the crate we're checking, add the `cargo-semver-checks` lints table:

```toml
# ...
[package.metadata.cargo-semver-checks.lints]
function_missing = { level = "warn", required-update = "minor" }
```

To configure other lints, just add them as additional entries to that table.  Note that it is not required to configure *both* lint level and required version update, and you can use the shorthand `lint_id = "level"` to configure just the lint level:

```toml
[package.metadata.cargo-semver-checks.lints]
function_missing = { level = "warn", required-update = "minor" }
trait_now_doc_hidden = "allow" # shorthand to set just lint level to `allow`
enum_variant_added = { level = "deny" } # leaves required-update as the default
function_changed_abi = { required-update = "minor" } # leaves level as default
```

This table should be placed in the current/new `Cargo.toml` manifest, not the baseline/old manifest.  (this defaults to `./Cargo.toml` in the directory that `cargo-semver-checks` was invoked on, and can be configured to a different path with the `--manifest-path` CLI option).

## configuring a workspace

It may also be helpful to configure these checks at the workspace level for multiple different crates in the same Cargo workspace. To do this, in the workspace root `Cargo.toml`, add a similar configuration table:

```toml
[workspace.metadata.cargo-semver-checks.lints]
function_missing = { level = "warn", required-update = "minor" }
```

Note that it is `workspace.metadata` and not `package.metadata`.  Then, to opt-in to these for each package that is being semver-checked, one of these keys to the *package* Cargo.toml:

```toml
[lints]
workspace = true
```

or 

```toml
[package.metadata.cargo-semver-checks.lints]
workspace = true
```

Either of these is acceptable to indicate to read the `[workspace.metadata.cargo-semver-checks.lints]` table for this package.  Using the `lints.workspace` key can be helpful to indicate this behavior for other linters (e.g., cargo and clippy), but will cause a cargo error if it is set and there is no `[workspace.lints]` table in the workspace Cargo.toml.  To solve this, we added the `package.metadata.cargo-semver-checks.lints.workspace` key for compatibility. Note that setting `workspace = false` is not valid configuration for either of these keys.  To indicate not to read workspace configuration for a crate, simply omit both keys entirely from the package's Cargo.toml.

### overriding workspace configuration

When `workspace = true` is set, it is possible to override individual lint configuration lines in a package.  For example, if we have in the *workspace* Cargo.toml:

```toml
[workspace.metadata.cargo-semver-checks.lints]
function_missing = { level = "warn", required-update = "minor" }
trait_now_doc_hidden = "warn"
```

and in the *package* Cargo.toml:

```toml
[package.metadata.cargo-semver-checks.lints]
workspace = true
function_missing = "deny"
trait_now_doc_hidden = { required-update = "major" }
```

Fields set in the package configuration override the workspace configuration.  Thus, the lint level for `function_missing` will be overridden to `deny`, and the workspace default `required-update` of `minor` will be used because it is not configured in the package.  Similarly, the lint level for `trait_now_doc_hidden` will be the workspace's `warn`, but the required update will be overridden in the package to a `major` version bump.

## limitations

Currently, the configuration can only be read from the new version of the checked crate's Cargo.toml.  If this manifest isn't used (for example, when using the `--current-rustdoc` to use a preexisting rustdoc output instead, it is not possible to configure lints.  Configuration through other means (e.g., CLI flags) could be added in the future if there is a strong demand for them.
