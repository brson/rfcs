- Feature Name: custom_test_runners, custom_test_frameworks
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Extend rustc's built-in test harness to use custom *test frameworks*,
for defining tests, as well as custom *test runners*, for running
tests independently of the framework. Test frameworks are standard
plugins, while runners are wired to frameworks with a simple but new
mechanism for overriding main and passing its initial state.

This RFC defines the interface between test runners and test
frameworks and how the compiler wires them together.

# Motivation
[motivation]: #motivation

Rust's built in test support is convenient but simple. There are lots
of features people want but we don't provide (and likely won't), such
as setup and teardown of common state between methods, or between
groups of methods (sometimes called 'test fixtures'), advanced DSLs,
property-based testing.

A number of test frameworks exist for Rust, and most of them are syntax
extensions or macros that 

# Detailed design
[design]: #detailed-design

Definitions used in this RFC:

* test framework - a plugin that generates a list of test functions at
  compile time test runner - a crate with a specially-annotated main
* function that accepts a list of test functions


## The test runner interface

```
mod __test {
  extern crate test (name = "test", vers = "...");
  fn main() {
    test::test_main_static(&::os::args()[], tests)
  }
  static tests : &'static [test::TestDescAndFn] = &[
    ... the list of tests in the crate ...
  ];
}
```



# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD?
