- Start Date: 2014-10-27
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

This proposal details a strategy for 'feature staging', which, in
conjunction with [release channels][1], allows development of
experimental language features and library APIs to proceed on the
'nightly' channel while providing strong stability guarantees in
stable releases.

[1]: http://discuss.rust-lang.org/t/rfc-impending-changes-to-the-release-process/508

# Motivation

The Rust compiler and accompanying libraries contain language features
and libraries that are considered experimental and subject to
change. We want to continue providing these features so that users
can test them and provide feedback, we want to be able to evolve
these features until they are satisfactory, and we also want to
provide strong stability guarantees to users of the language.

Precedent from other compilers, including GHC, which has feature
gates used indiscriminately by libraries, and gcc, which has numerous
legacy command-line options to change compiler behavior that must be
supported by other compilers (clang) for compatibility, strongly
indicates that any feature provided by the compiler, even if not
intended for general use, is at risk of becoming 'de-facto' stable
through wide adoption, restricting our ability to evolve the language
as intended.

Certain existing features in Rust must not be stabilized in their
present form. Perhaps most notably, the plugin architecture exposes
the entire compiler to plugin authors, presenting a scenario in which
either plugins may break on every release or the internal compiler
APIs themselves must be frozen, hampering evolution of the entire
language.

# Detailed design

In builds of Rust distributed through the beta and stable release
channels, it is impossible to turn on experimental language features
by writing the `#[feature(...)]` attribute or to use API's *from
libraries distributed as part of the main Rust distribution* tagged
with either `#[experimental]` or `#[unstable]`. This is accomplished
primarily through three new lints, `experimental_features`,
`staged_unstable`, and `staged_experimental`, which are set to 'allow'
by default in nightlies, and 'forbid' in beta and stable releases.

The `experimental_features` lint simply looks for all 'feature'
attributes and emits the message 'experimental feature'.

The `staged_unstable` and `staged_experimental` behave exactly like
the existing `unstable` and `experimental` lints, emitting the message
'unstable' and 'experimental', except that they only apply to crates
marked with the `#[staged_api]` attribute. If this attribute is not
present then the lints have no effect.

All crates in the Rust distribution are marked `#[staged_api]`.
Libraries in the Cargo registry are not bound to participate in
feature staging because they are not required to be
`#[staged_api]`. Crates maintained by the Rust project (the rust-lang
org on GitHub) but not included in the main Rust distribution are not
`#[staged_api]`.

The decision to set the feature staging lints is driven by a new field
of the compilation `Session`, `disable_staged_features`. When set to
true the lint pass will configure the three feature staging lints to
'forbid', with a `LintSource` of `ReleaseChannel`. Once set to
'forbid' it is not possible for code to programmaticaly disable the
lint. When a `ReleaseChannel` lint is triggered, in addition to the
lint's error message, it is accompanied by the note 'this feature may
not be used in the {channel} release channel', where `{channel}` is
the name of the release channel.

In feature-staged builds of Rust, rustdoc sets
`disable_staged_features` to *`false`*. Without doing so, it would not
be possible for rustdoc to successfully run against e.g. the
accompanying std crate, as rustdoc runs the lint pass. Additionally,
in feature-staged builds, rustdoc does not generate documentation for
experimental and unstable APIs for crates with the `#[staged_api]`
attribute.

With staged features disabled, the Rust build itself is not possible,
and some portion of the test suite will fail. To build the compiler
itself and keep the test suite working the build system activates
a hack via environment variables to disable the feature staging lints,
a mechanism that is not be available under typical use. The build
system additionally includes a way to run the test suite with the
feature staging lints enabled, providing a means of tracking what
portion of the test suite can be run without invoking experimental
features.

The prelude causes complications with this scheme because prelude
injection presently uses two feature gates: globs, to import the
prelude, and phase, to import the standard `macro_rules!` macros. In
the short term this will be worked-around with hacks in the
compiler. It's likely that these hacks can be removed before 1.0 if
globs and `macro_rules!` imports become stable.

# Drawbacks

The major risk in feature staging is that, at the 1.0 release not
enough of the language is available to foster a meaningful library
ecosystem around the stable release. While we might expect many users
to continue using nightly releases with or without this change, if the
stable 1.0 release cannot be used in any practical sense it will be
problematic from a PR perspective. Implementing this RFC will require
careful attention to the libraries it affects.

Recorgnizing this risk, we must put in place processes to monitor the
compatibility of known Cargo crates with the stable release channel,
using evidence drawn from those crates to prioritize the stabilization
of features and libraries. The details of this process are out of
scope for this RFC.

Syntax extensions, lints, and any program using the compiler API's
will not be compatible with the stable release channel at 1.0 since it
is not possible to stabilize `#[plugin_registrar]` in time. Plugins
are very popular. This pain will partially be alleviated by a proposed
[Cargo] feature that enables Rust code generation.

[Cargo]: https://github.com/rust-lang/rfcs/pull/403

# Alternatives

Leave feature gates and experimental APIs exposed to the stable
channel, as precented by Haskell, web vendor prefixes, and node.js.

Make the beta channel a compromise between the nightly and stable
channels, allowing some set of experimental features and APIs. This
would allow more projects to use a 'more stable' release, but would
make beta no longer representative of the pending stable release.

# Unresolved questions

The exact method for working around the prelude's use of feature gates
is undetermined. Fixing [#18102] will complicate the situation as the
prelude relies on a bug in lint checking to work at all.

[#18102]: https://github.com/rust-lang/rust/issues/18102

We might leverage the test suite to greater or lesser degrees to help
us track metrics about what code can be used with the stable
channel. Details undecided.

Rustdoc disables the feature-staging lints so they don't cause it to
fail, but I don't know why rustdoc needs to be running lints. It may
be possible to just stop running lints in rustdoc.
