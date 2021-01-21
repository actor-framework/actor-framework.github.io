---
layout: post
title:  "Benchmarking 0.18"
author: dominik
tags: Performance Benchmark
---

# Introduction

CAF 0.18 is out!

More than a year has passed from designing and prototyping the new messaging
layer to cutting the release that implements the changes. In our announcement on
breaking changes from 11/2019, we quoted performance considerations as one of
our primary motivations for implementing the new messaging layer:

> Over the course of the last years of developing CAF, we came to understand
> `message` as a `variant`-like type (...), but the implementation (...) comes
> at a cost. Obvious costs through virtual dispatch, but also runtime overhead
> for matching message handler signatures to the content of incoming messages.
> Further, because a message can essentially hold anything, CAF has to include a
> lot of meta data in each message on the wire. This increases size overhead on
> the wire but also makes deserializing messages costly.

Prior to CAF 0.18, we relied on `std::type_info` and held a run-time generated
table that users filled with custom types by calling
`actor_system_config::add_message_type` prior to initializing the
`actor_system`.

With the new type ID blocks, users instead have to list all their types
statically. This also comes with the benefit of allowing CAF to check whether a
type was listed as a legal message type. Not calling
`actor_system_config::add_message_type` was a common source of error, especially
when introducing new message types to an existing software. With the new type ID
blocks, you still have to initialize the run-time lookup table (manually or by
passing the type ID to `CAF_MAIN`) but this is robust under refactoring: adding
a type to the block also updates the initialization logic.

The new type IDs in CAF allow us to reduce the amount of meta information
drastically. A `caf::message` simply points to an array of 16-bit values that
encode the run-time types of all elements. On the wire, CAF no longer prefixes a
message with the full type names as string but again with a list of 16-bit
integers. The new implementation for `caf::message` also uses a flat memory
layout instead of the previous tree structure.

So how much performance improvement, if any, can we expect after porting an
application to 0.18? To answer this question, we have implemented a set of micro
benchmarks. The full source code is available
[online on GitHub](https://github.com/actor-framework/micro-benchmarks).

# The Benchmarks

Our first benchmark set is called `message_creation`. It looks at the run-time
cost of creating messages. CAF has two APIs to do this: pass all elements at
once to `make_message` or create a message dynamically using a
`message_builder`:

```cpp
BENCHMARK_F(message_creation, make_message)(benchmark::State& state) {
  for (auto _ : state) {
    auto msg = make_message(size_t{0});
    benchmark::DoNotOptimize(msg);
  }
}

BENCHMARK_F(message_creation, message_builder)(benchmark::State& state) {
  for (auto _ : state) {
    message_builder mb;
    message msg = mb.append(size_t{0}).move_to_message();
    benchmark::DoNotOptimize(msg);
  }
}
```

The next two benchmark sets `pattern_matching` and `or_else` measure how long it
takes to dispatch a message to a `behavior`. The only difference between the two
sets is that the former creates the behavior form a single list of lambda
expressions, while the latter wraps each lambda into an individual `behavior`
before gluing all of them together via `or_else`. The messages as well as the
behaviors are created once during setup before running the benchmark loop. The
implementation for these benchmarks involves setting up fixtures, so we omit
them here for brevity.

The last benchmark set is called `serialization` and it measures the run-time
cost of saving or loading builtin and custom types. As custom types, the
benchmark uses a simple POD called `foo` and a slightly more "advanced" type
`bar` that includes the custom type `foo` as a member variable to force CAF to
recursively call `inspect`:

```cpp
struct foo {
  int32_t a;
  int32_t b;
};

struct bar {
  foo a;
  std::string b;
};

#if CAF_VERSION >= 1800

template <typename Inspector>
bool inspect(Inspector& f, foo& x) {
  return f.object(x).fields(f.field("a", x.a), f.field("b", x.b));
}

template <typename Inspector>
bool inspect(Inspector& f, bar& x) {
  return f.object(x).fields(f.field("a", x.a), f.field("b", x.b));
}

#else // CAF_VERSION >= 1800

template <typename Inspector>
typename Inspector::result_type inspect(Inspector& f, foo& x) {
  return f(x.a, x.b);
}

template <typename Inspector>
typename Inspector::result_type inspect(Inspector& f, bar& x) {
  return f(x.a, x.b);
}

#endif // CAF_VERSION >= 1800
```

This benchmark also creates the messages once before running the benchmark
loops. We omit the implementations details here also for brevity. The sources
are just
[one click away](https://github.com/actor-framework/micro-benchmarks/tree/b7d063b0d884ab9c210221c5d101e4a4797b2b7e/src),
though.

# Setup

Hardware in use:

- 3.6 GHz 10-Core Intel Core i9
- 64 GB 2667 MHz DDR4
- CPU Caches:
  + L1 Data 32 KiB (x10)
  + L1 Instruction 32 KiB (x10)
  + L2 Unified 256 KiB (x10)
  + L3 Unified 20480 KiB (x1)

Software versions:

- actor-framework/micro-benchmarks: `b7d063b0d884ab9c210221c5d101e4a4797b2b7e`
- actor-framework/actor-framework:
  - CAF 0.16.5 is tag `0.16.5`
  - CAF 0.17.6 is tag `0.17.6`
  - CAF 0.18.0 is commit `8d27730124440c5c7422126a222b1bf2a0262e3f` (slightly
    before the final release tag)

# Results

Without further ado, here are the results for the micro benchmarks using the
last three major release versions. All values are wall-clock time in
*nanoseconds* as reported by Google Benchmark.

| Benchmark                            |  CAF 0.16.5 | CAF 0.17.6 | CAF 0.18 |
|--------------------------------------|------------:|-----------:|---------:|
| `message_creation/make_message`      |        57.3 |       57.3 | 53.5     |
| `message_creation/message_builder`   |         258 |        248 |	339      |
| `pattern_matching/make_message`      |         233 |        219 |	103      |
| `pattern_matching/message_builder`   |         239 |        234 |	102      |
| `or_else/make_message`               |         364 |        347 |	168      |
| `or_else/message_builder`            |         348 |        346 |	167      |
| `serialization/save_foo_binary`      |         143 |        136 |	110      |
| `serialization/save_bar_binary`      |         174 |        174 |	127      |
| `serialization/save_msg_2int_binary` |         247 |        237 |	138      |
| `serialization/load_foo_binary`      |        47.8 |       31.0 |	8.44     |
| `serialization/load_bar_binary`      |        84.7 |       55.5 |	16.8     |
| `serialization/load_msg_2int_binary` |         627 |        566 |	266      |
|                                      |             |            |          |

Creating messages using `make_message` is slightly faster than before, due to
the simpler (flat) data structure used internally as of 0.18. However, the same
design makes the `message_builder` slower than before, because once all values
are added to the builder, it has to allocate a new block of memory and move or
copy everything into place.

Selecting a matching handler from a `behavior` is much quicker than before with
a speedup of > 2 compared to earlier versions of CAF.

The serialization components also show significant performance improvements.
Saving values to a `binary_serializer` is now 20-70% faster, because CAF has to
generate less data. The biggest win comes when deserializing objects from a
`binary_deserializer`. There is less data to read and string lookups became
simple offset lookups due to the new type IDs.

# Remarks

When browsing the micro benchmarks repository, you might notice one additional
benchmark not covered here: `actors/spawn_and_await`. This benchmark spawns an
actor that does nothing and then waits until it terminated. The values we had
measured in our performance study where 41355ns, 41142ns and 42618ns for CAF
0.16.5, 0.17.6 and 0.18 respectively. This result fluctuates more than the
others. We did not focus on this result, because it measures many components in
CAF all at once.

If you had followed our past benchmarking articles or read some of our research
papers, you may also remember the benchmark suite consisting of `mixed_case`,
`actor_creation`, and `mailbox_performance`. For the sake of completeness, we
also recorded values for those. This time, all values in *seconds*:

| Benchmark                            |  CAF 0.16.5 | CAF 0.17.6 | CAF 0.18 |
|--------------------------------------|------------:|-----------:|---------:|
| `mixed_case`                         |        22.8 |       26.8 | 24.2     |
| `actor_creation`                     |         2.9 |        1.3 | 0.7      |
| `mailbox_performance`                |        34.9 |       35.2 | 33.7     |
|                                      |             |            |          |

Results shown in the table above are the average of ten runs. None of these
benchmarks do any networking and thus no serialization. We included them mostly
as a sanity check here to make sure CAF 0.18 does not introduce performance
regressions. You can also find these three benchmarks (among others)
[online on GitHub](https://github.com/actor-framework/benchmarks). The exact
commit used here is `383b745a4ad18b22a48166a048cb34e1bce40aaf`.

# Conclusion

CAF 0.18 introduced a new type inspection API and static type IDs for
user-defined types. These two changes are most obvious when switching to the new
release, because they require code changes.

Overall, we are very happy with the performance we get from the new messaging
layer. Keep in mind that micro-benchmark results for serialization with speedup
of 2 or more do not translate to a speedup of that magnitude for a distributed
CAF application. Serialization is only one fraction of the total work.

Aside from any performance gains, we are most excited about new capabilities
opening up as a result of the new type inspection and messaging layer. The new
type inspection API passes more information to CAF (especially field names),
which allows us to read and write more data formats such as JSON. Using more
compact meta information on the wire and avoiding dependencies on
`std::type_info` also allows us to tune CAF for resource-constrained devices to
enable more IOT and edge computing use cases.
