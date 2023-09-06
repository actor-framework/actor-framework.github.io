---
layout: post
title:  "New Unit Testing Framework"
author: dominik
tags: API
---

# Unit Testing Reloaded

As teased in our last
[roadmap update](https://www.actor-framework.org/blog/2021-05/roadmap-and-community-update),
CAF is getting a new unit testing framework.

We are happy to announce that the new framework will be available as part of the
upcoming CAF 0.19.3 release. We have already started to migrate some of our own
tests to the new framework and are very happy with the results so far.

Before starting to implement the new framework, we evaluated several existing
options as well as the possibility to extend the previous framework. However,
ultimately we decided to start from scratch and build a new framework that
tightly integrates with CAF and its type system while also providing a modern,
easy-to-use API.

## Features

For our new framework, we wanted to provide a feature set that is comparable to
other popular unit testing frameworks such as
[doctest](https://github.com/doctest/doctest) or
[Catch2](https://github.com/catchorg/Catch2) while also providing a seamless
integration with CAF. For example, the unit testing framework will be able to
automatically render any type that provides an
[inspect](https://actor-framework.readthedocs.io/en/stable/TypeInspection.html)
overload.

Unlike our previous framework as well as most other frameworks, we use actual
C++ functions for `check` and `require` statements instead of macros. We
propagate meta information such as the file name and line number to the
functions via a backport of C++20's
[`source_location`](https://en.cppreference.com/w/cpp/utility/source_location)
class. However, unlike frameworks such as [μt](https://github.com/boost-ext/ut),
we do still use macros for structuring the test code to rely less on lambda
expressions. We strive to strike a good balance between using modern C++
features and providing a straightforward API.

In a nutshell, our new testing framework provides these features:

- Test cases with any number of nested sections.
- Check and require functions for verifying boolean expressions.
- BDD-style scenario testing with *given*, *when*, *then* blocks.
- BDD-style scenario outlines (scenario templates).
- Single-file tests as well as multi-file tests with test suites.
- Backwards compatible CLI interface for running tests.
- Lazy generation of test output (only prints failed tests by default).
- Integration with type ID blocks (similar to `CAF_MAIN`).
- Pre-build test fixtures that provide DSLs for deterministic testing of CAF
  actors, flows, networking, and more.

To get a full overview of the new framework, please refer to the
[readme](https://github.com/actor-framework/actor-framework/blob/master/libcaf_test/README.md)
of the test module.

# Examples

Sections form a tree structure and CAF runs one branch at a time:

```cpp
TEST("each run starts with fresh local variables") {
  auto my_int = 0;
  SECTION("block 1 reads my_int as 0") {
    check_eq(my_int, 0);
    my_int = 42;
    check_eq(my_int, 42);
  }
  SECTION("block 2 also reads my_int as 0") {
    check_eq(my_int, 0);
  }
}
```

Like sections, `WHEN`-blocks also run one at a time:

```cpp
SCENARIO("each run starts with fresh local variables") {
  GIVEN("a my_int variable") {
    auto my_int = 0;
    WHEN("entering a WHEN block") {
      THEN("the local variable has its default value") {
        check_eq(my_int, 0);
        my_int = 42;
        check_eq(my_int, 42);
      }
    }
    WHEN("entering another WHEN block") {
      THEN("previous writes to the local variable are gone") {
        check_eq(my_int, 0);
      }
    }
  }
}
```

Outlines allow to run the same scenario with different parameters:

```cpp
OUTLINE("eating cucumbers") {
  GIVEN("there are <start> cucumbers") {
    auto start = block_parameters<int>();
    auto cucumbers = start;
    WHEN("I eat <eat> cucumbers") {
      auto eat = block_parameters<int>();
      cucumbers -= eat;
      THEN("I should have <left> cucumbers") {
        auto left = block_parameters<int>();
        check_eq(cucumbers, left);
      }
    }
  }
  EXAMPLES = R"(
    | start | eat | left |
    |    12 |   5 |    7 |
    |    20 |   5 |   15 |
  )";
}
```

Testing actors in a deterministic fashion is easy with the new framework:

```cpp
WITH_FIXTURE(caf::test::fixture::deterministic) {

TEST("the deterministic fixture provides a DSL for testing actors") {
  auto count = std::make_shared<int32_t>(0);
  auto worker = sys.spawn([count] {
    return caf::behavior{
      [count](int32_t value) { *count += value; },
    };
  });
  caf::scoped_actor self{sys};
  anon_send(worker, 1);
  self->send(worker, 2);
  anon_send(worker, 3);
  SECTION("expect() checks for required messages") {
    expect<int32_t>().to(worker);
    check_eq(*count, 1);
    expect<int32_t>().to(worker);
    check_eq(*count, 3);
    expect<int32_t>().to(worker);
    check_eq(*count, 6);
  }
  SECTION("expect() optionally matches the content of the next message") {
    expect<int32_t>().with(1).to(worker);
    expect<int32_t>().with(2).to(worker);
    expect<int32_t>().with(3).to(worker);
    check_eq(*count, 6);
  }
  SECTION("expect() optionally matches the sender of the next message") {
    expect<int32_t>().from(nullptr).to(worker);
    expect<int32_t>().from(self).to(worker);
    check_eq(*count, 3);
  }
}

} // WITH_FIXTURE(caf::test::fixture::deterministic)
```

## Implementation Status

The basic functionality of the new framework is already implemented and we are
currently in the process of migrating some of our own tests to the new
framework. However, the fixture for the deterministic testing of actors is still
lacking some features and we are also planning to add fixtures for testing flows
and networking.

As always, feedback is most welcome! If you have any suggestions or ideas for
improvements, please let us know!
