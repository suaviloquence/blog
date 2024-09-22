+++
title = "Google Summer of Code Final Writeup"
date = 2024-09-23

[taxonomies]
tags=["gsoc24", "rust"]

[extra]
# issue = 7
+++

Summer is officially over now, and now so is my [Google Summer of Code project](@/gsoc-24-intro.md) for this year.  This was a really great experience for me, and I'm super grateful to have had this opportunity.  I had the great help of being mentored by the maintainer of the project [Predrag Gruevski](https://predr.ag/), which I couldn't have asked for a better mentor for this.  In this blog post, I'll talk about the work I did, reflections on my experience, and what the future looks like:

## the work

The title of my GSoC project that I wrote at the beginning is "adding lint-level configuration to `cargo-semver-checks`."  This means adding the ability for users to configure each individual check of the semantic versioning linter [`cargo-semver-checks`](https://github.com/obi1kenobi/cargo-semver-checks/). [^project-info] Adding this configuration granularity will let many more Rust crates adopt `cargo-semver-checks` to automatically help prevent breaking the semantic versioning guarantees, which helps the whole Rust ecosystem make more fearless upgrades according to SemVer.  

### overrides

To configure each of `cargo-semver-checks`'s 90+ lints, both by lint level/severity and required SemVer update, I first added a [default lint level field (#787)](https://github.com/obi1kenobi/cargo-semver-checks/commit/a8083aabf4ae46167b435c7f4aba3425fbf463c5) to each lint, then added a [new API (#788)](https://github.com/obi1kenobi/cargo-semver-checks/commit/393691c79dc9a70cdc557fd4b3b24e988b0ec307) to configure overriding each lint's fields.  I had to design a system that supported multiple levels of precedence, with the ability to configure lint level, required update, or both for a given lint at each of these precedence levels, so it took some careful design and adding plenty of unit tests.

### the `[lints]` table

After posting my initial blog post, the feedback I received from community members was very helpful.  Initially, I had planned to start exposing the configuration interface as a series of CLI flags (like `rustc`'s `--warn <name>`).  In this discussion with my mentor and community members like [epage](https://github.com/epage), it started to look like a better idea to work on the `[lints]` table in the Cargo manifest first, and reevaluate the need for CLI flags later on.  

After this, I implemented the logic to read a table similar to the Cargo `[lints]` table from manifests in [#799](https://github.com/obi1kenobi/cargo-semver-checks/commit/f8f89ecfa7f1e60b343e7b07beaf415393f244f8), using the `[package.metadata.cargo-semver-checks.lints]` (it's currently not valid for tools like `cargo-semver-checks` to use the `[lints]` table directly), and in [#800](https://github.com/obi1kenobi/cargo-semver-checks/commit/4e0b92629a96413f875035d6d537a8409a90baf1), I implemented the logic to use the configured required update, and added tests for the many different interactions, especially with both workspace and package configuration.

After some minor fixes in [#804](https://github.com/obi1kenobi/cargo-semver-checks/commit/ee0ce6df438aee7be50fe67041481a20c4257d3b), and [#806](https://github.com/obi1kenobi/cargo-semver-checks/commit/4586521f8976947ea45a7e40c2218cbf7c051c9a), I finished the logic for using the configured lint level in [#805](https://github.com/obi1kenobi/cargo-semver-checks/commit/fe1830f2e550bf2d82ccf1b6705571da12a74fbc).  THis was much more design-intensive than using the configured required update, because this was also introducing the concept of warning-level checks to `cargo-semver-checks`, and I spent a lot of time with my mentor designing how a warning should look like in `cargo-semver-checks`'s interface, and how warnings should interact with errors.

In [#808](https://github.com/obi1kenobi/cargo-semver-checks/commit/137790a595ea8ae0166e5a0b39ac1f60ee23b044) and [#809](https://github.com/obi1kenobi/cargo-semver-checks/commit/a72aa1388af5e3766bc93bd03de43260ed8f048e) I added some future-proofing tests and comments to help with maintainability.

### compatibility with `cargo`

A goal of `cargo-semver-checks` as a project is to be as compatible with `cargo` as possible and eventually be merged with the tool.  After posting another update on `cargo-semver-checks`'s new configurability, part of the feedback to this was that the current lints table would not be quite compatible with cargo's.  In [#811](https://github.com/obi1kenobi/cargo-semver-checks/commit/43678899bcc9f748101e9226fb4995a54555d463), [#812](https://github.com/obi1kenobi/cargo-semver-checks/commit/a4f745f94dbf23ed9cf13bfe72b9cad875de4d3c), and [#813](https://github.com/obi1kenobi/cargo-semver-checks/commit/363754a3811d3fdf0ce5ac115723ea2ecf74558e), I added the functionality to mimic `cargo` and only read workspace configuration if a key is explicitly set in the package, and added tests (including making sure that `cargo-semver-checks` denies the `lints.workspace = false` key, which is insidiously ian invalid state).  

### documenting

At this point, the initial core configuration was ready, so I added a section to the README that was a guide on how to use this new feature in [#826](https://github.com/obi1kenobi/cargo-semver-checks/commit/319943604b02fa814def31163a418326ea7cb460) and [#829](https://github.com/obi1kenobi/cargo-semver-checks/commit/c11ec6d02b4cea17209be9239ff8c1bb6ff025ad).  I wrote a lot of documentation this summer, both developer-facing and user-facing, and it's one of the things I'm proud of.  To me, documentation is often the first impression of a project, and it can make or break a user's experience, so polishing it is super important.

### a detour?

Here in the project, there was kind of a lull, and I was working on polishing tests and documentation.  However, one thing I was noticing was that it was kind of hard and involved to write a full integration test case.  My mentor suggested that I try to improve the testing infrastructure for `cargo-semver-checks`, and I thought that was a great idea.  In [#846](https://github.com/obi1kenobi/cargo-semver-checks/commit/329c42f1cfadba23454d881273f63df40a6291bc), I added the foundation to be able to write integration tests using the `insta` framework.  I also spent some time benchmarking to figure out what kind of test (i.e., `lib test`, `bin test`, or, confusingly, `test` binary) would be most helpful to access the right parts of `cargo-semver-checks` to test while having the fastest possible compile times for better iteration and design.  Here, I also wrote a lot of documentation for future contributors to have the best experience when writing new tests.

After adding this test infrastructure, I wrote some tests that pinned down `cargo-semver-checks`'s behavior for different edge cases, which documented this behavior, prevented any unwanted regressions, and would be easy to change if we decided in the future we wanted this behavior to be different, in [#847](https://github.com/obi1kenobi/cargo-semver-checks/commit/0ed482776ba158df55ff3651aaa95019249e04b4), [#859](https://github.com/obi1kenobi/cargo-semver-checks/commit/62a3c75215399f9c5d853b7dad24147af6a01f42), [#866](https://github.com/obi1kenobi/cargo-semver-checks/commit/c53ff88cd6d6d4e0631c4405d5f8c0da562d8218), and [#882](https://github.com/obi1kenobi/cargo-semver-checks/commit/6e5d78acf899dbb232060cfc525b7444a3d2f7e5).  In writing these tests, I found the opportunity to polish in fix parts of the test infrastructure, like in [#865](https://github.com/obi1kenobi/cargo-semver-checks/commit/eb5c87dac9ce2c51d054272d6ba902cc4121bc3f), [#867](https://github.com/obi1kenobi/cargo-semver-checks/commit/8898271377a3931925ba38518b1995c96107795f), [#881](https://github.com/obi1kenobi/cargo-semver-checks/commit/5b35f9f3d66ea76a6dd87fa348fabd12224a30c3), and [#883](https://github.com/obi1kenobi/cargo-semver-checks/commit/c2dfe22505975f3092820e4464ccb2774e87928c), as well as [#913](https://github.com/obi1kenobi/cargo-semver-checks/commit/7b7a518150b7ff2701c40a61829896a2cd05898b) later on.

### new features!

At this point, the [user demand](https://github.com/obi1kenobi/cargo-semver-checks/issues/827) for CLI-configurable lints when there was already another avenue to configure them was not that strong, so it didn't seem like the best way forward to continue with my original proposed idea of adding CLI flags here.  The project started to go off the rails a little, but in a very good way.  Predrag suggested that I work on a new feature to generate witness programs, which are buildable examples of the downstream breakage of agiven breaking change, which prove that a SemVer guarantee would be broken with this change.  As a fan of testing and correctness, I got to work on this feature. 

Adding witnesses is a big feature, so we wanted the ability to mark a feature as unstable during its development.  Following the design of other tools in the Rust ecosystem like Cargo, I worked on the ability to have unstable feature flags and CLI options in [#896](https://github.com/obi1kenobi/cargo-semver-checks/commit/eb0c7713c61a3bc9de5a872e3327eaa2e59c462e), and I added some various consistency improvements in [#897](https://github.com/obi1kenobi/cargo-semver-checks/commit/a39dc2b46bb52aadfb02d067ea3904719ec89fe8), [#899](https://github.com/obi1kenobi/cargo-semver-checks/commit/cf060a624fbac0ffc6332319d9b7c560330a8e83), and [#919](https://github.com/obi1kenobi/cargo-semver-checks/commit/3352f525554a825745465ba02f41af2200c37a73).

I added the start of the witness feature in [#893](https://github.com/obi1kenobi/cargo-semver-checks/commit/a056a5a743370612109ea32a797859a2fea8a210), and improved the contributor experience and documentation for adding new witnesses in [#935](https://github.com/obi1kenobi/cargo-semver-checks/commit/1bd25e6348a528871702af724edfbfc93e08130) and [#933](https://github.com/obi1kenobi/cargo-semver-checks/commit/0f67fc4d92c310cc9ff3ab617ac36d058fb8aadc).

Work on witnesses is currently unfinished ongoing, and this is one of the things I will keep working on as GSoC finishes.

### lint groups

One of the stretch goals in my original project was adding configurable lint groups in addition to configuring individual lints, and this feature was seeming more useful than adding CLI configuration flags.  I worked on a [design draft](@/lint-groups-cli-draft.md), focusing on how a user would use these features.  I started the work by reading the `priority` field to the `[lints]` table in [#932](https://github.com/obi1kenobi/cargo-semver-checks/commit/68ee754452a74bc987bc98fdf29443efffd08edf) to avoid configuration conflicts by defining precedence, and I have drafts of the logic to have lint group configuration, so expect to see lint groups in `cargo-semver-checks` pretty soon.

---

...and that's it.  I merged 40 commits during GSoC which feels like a lot, but also not a lot, because each of those commits is squashed and many of them have several parts/squashed commits and (quite in-depth) discussions in the pull request, so the commit history does not tell the full story.

## thanks

About those discussions: I could not get this far in the blog post without recognizing my mentor, Predrag Gruevski. His attention to detail (down to the number of spaces after a period), thoughtfulness, kindness, and careful consideration of every line of code I merged into `cargo-semver-checks` (and the so many that I didn't) encouraged me to design, implement, and think about the code I submitted from different perspectives and as part of a complex system.  This was so helpful in shaping the way I think about designing features and contributing to open source, and I can't thank him enough.

Additionally, the community around the Rust GSoC program and the Rust project in general was a super great environment to be a part of.  Special thanks to the GSoC admin Jakub Beránek, maintainers and contributors like Ed Page and Scott Schafer, everyone else working on a Rust GSoC project this summer - it was such a great experience to see updates and how everyone's project progressed over the summer, and the whole open source community.  I can't overstate what a welcoming experience this has been, and I'm definitely going to keep being a part of the community in the future.


### and an apology

[The first PR](https://github.com/obi1kenobi/cargo-semver-checks/pull/784) I opened as a part of my GSoC project was... a little overzealous.  It changed hundreds of lines of code, affected multiple different systems, added multiple different features, and was overall a huge set of changes to review.  My mentor (very nicely and diplomatically) suggested I might break up the changes into smaller PRs.  This was a great learning experience for me to reflect on my git hygiene and how to make my changes easier and clearer to review, which is something I worked on and developed over the summer.  I'm grateful to have had this opportunity to grow, and I do apologize for this first pull request ;).

## what's next

This is certainly not the end of my work on `cargo-semver-checks`.  Right now, I have three projects I'm working on: adding the functionality for lint groups, keeping working on adding witnesses, and working on a [refactor of the command line output](https://github.com/obi1kenobi/cargo-semver-checks/pull/939) of `cargo-semver-checks`.  With the new features, `cargo-semver-checks` outgrew its old interface a little bit, and we're working on exploring making it more consistent with the output of `rustc` and `clippy`.

Additionally, I'm going to keep working on [some of my own projects](https://github.com/suaviloquence/scrapelect/), and I'm also planning on contributing to some other Rust open-source projects now that GSoC is over.  I'd like to keep posting about my journey on this blog, and I made a [Mastodon](https://fosstodon.org/@m_carr) for interacting with the open-source community.  (and I'm always looking for cool people to follow!)

Thanks again to everyone who made this summer possible!

[^project-info]: For more information about the project, I wrote a [blog post](@/gsoc-24-intro.md) describing the problem I set out to solve this summer.
