---
layout: post
title:  "Announcing Version 1.0"
author: dominik
tags: API Roadmap
---

# Towards a New Decade

Today, we are thrilled to announce the release of CAF version 1.0. This is a
major milestone for us and we are excited to share it with you.

CAF has been in beta for over a decade. During that time, we have revised the
design multiple times, continuously added features and received valuable
feedback from our users. We are very grateful for the comments, suggestions and
bug reports we have received over the years. Be it on GitHub, in chat, via email
or in private conversations, your feedback has been invaluable to us.

For this milestone release, we have asked ourselves how CAF would look like if
we started from scratch today and how we want to evolve CAF over the next
decade. Of course, we did not throw away everything we have built so far.
However, we have thought of all the lessons we have learned, considered the
community feedback we have received and discussed our experience of teaching CAF
to new users.

Our goal for CAF version 1.0 is to provide a solid foundation for the next
decade. We want to make CAF more accessible to new users while keeping the
flexibility and power that our experienced users have come to appreciate.
Ultimately, we want to make CAF easier to learn and teach, with a coherent API
design. Whether you are using CAF for cloud applications, micro services, edge
computing or the IoT, we strive to make CAF a great choice for building
distributed systems.

A secondary, but nonetheless important, goal is to provide more stability in the
API and the ABI. We want to make it easier to package CAF for distributions
while allowing us to continue to evolve the framework. To this end, we have
carefully reviewed the API and have deprecated several features that we consider
overly complex, error-prone or not in line with the overall design of CAF.

When looking at the changelog, you will notice that version 1.0 has more
deprecations than any previous release and that we have touched APIs that are at
the core of CAF. We have recently introduced the
[new mail API](https://www.actor-framework.org/blog/2024-05/simplifying-send)
that replaces the old messaging API. Today, we would like to use this blog post
to discuss further changes we have made and the reasons behind them.

## Deprecating `aout`

The `aout` utility has been part of CAF for a long time. It provides a way to
print to the console from multiple actors in parallel without interleaving
output. However, the design of `aout` tries to mimic `std::cout`. With
`std::format` on the horizon, we have decided to deprecate `aout` in favor of a
printing API that is more in line with modern C++.

For printing to the console, CAF now provides a `println` member function that
exists on actors and the actor system. The syntax is similar to `std::format`
and allows for type-safe printing of arbitrary values. When building CAF in
C++20 mode, `println` will use `std::format` under the hood. In C++17 mode, it
will use a custom implementation that supports only a subset of the formatting
options. In both cases, CAF will automatically pick up `inspect` overloads to
render custom types.

## Simpler Monitoring

When implementing event-based actors, a callback-based API is much more natural
than calling `monitor` and then providing a special handler someplace else.
Thus, we have deprecated the single-argument `monitor` function on event-based
actors in favor of a new overload that takes two arguments: the actor to monitor
and a callback function. The new overload also returns a `disposable` object
that can be used to cancel the monitoring.

Alongside this change, we have also deprecated the `set_down_handler` function,
since it becomes obsolete when using the new `monitor` overload. The `monitored`
spawn flag has been deprecated for the same reason.

## Better Idle Timeout Handling

Binding the idle timeout to the actor behavior has been a source of confusion
for many users. Consider this example:

```c++
caf::behavior my_actor(caf::event_based_actor* self) {
  return {
    // ... handlers ...
    caf::after(10s) >> [] { /* ... */ }
  };
}
```

Without looking at the documentation, it is not clear whether the timeout is
triggered once or repeatedly. Furthermore, it is unclear what kind of reference
will CAF keep to the actor. Will it keep the actor alive or will it only keep a
weak reference? The answer is that CAF will keep a strong reference to the actor
and trigger the timeout repeatedly. This is not always what users expect or
want. To trigger a timeout only once, users have to call `self->become()` in the
callback to install a new behavior that does not include the timeout.

To address this issue, we have deprecated this use of `after` and introduced a
new `set_idle_handler(timeout, ref_type, repeat_type, callback)` function, where
`timeout` is a duration, `ref_type` is either `strong_ref` or `weak_ref` and
`repeat_type` is either `once` or `repeat`. The `callback` is a function that
will be called when the timeout triggers. This new approach separates the idle
timeout from the behavior and adds much clarity to the code.

Note that nothing changes for blocking actors. They still pass `after` to
`receive` as before (it simply configures how long the function may block). The
new function is only relevant for event-based actors.

## Fewer Special Handlers

In the past, CAF has provided special handlers for certain message types such as
`exit_msg` as well as a catch-all handler for unexpected messages.

This design moves message handling logic away from the behavior, making it
harder to understand the actor's behavior at a glance. It also makes it harder
to reason about an actor in general, as the behavior is now spread across
multiple functions.

To address this issue, we have deprecated the special handlers. Instead, actors
can now simply handle these types of messages in their regular behavior and
provide a message handler for `message` as a catch-all.

## Deprecating the Legacy Unit Testing Framework

With CAF 1.0, we now include a fully-featured version of the new unit testing
framework that we
[have been working on](https://www.actor-framework.org/blog/2023-09/new-unit-testing-framework).
Our team has been using this framework for a while now and we are very happy
with it. We have ported all of our tests to the new framework and are
deprecating the old testing framework as a result. The old headers are still
available, but will be removed in version 2.0.

## Simplifying Spawn

CAF provides multiple ways to spawn actors. We have supported and promoted the
use of state-based actors for a while, but there have been several ways to
implement them as well. The feedback we have received from users is that the
current API is too complex and that it is not always clear which function to
use.

To address this, we have streamlined the API for spawning actors and there are
only two ways we recommend to spawn actors now: as a function or using a state
class. For the latter, we have already shipped the new `actor_from_state`
utility in CAF 0.19.5.

All of our examples and guides have been updated to reflect this.

However, we have decided against deprecating the old spawn functions right away.
This release already has a lot of deprecations in fundamental parts of CAF and
we want to give users more time to adjust to the new API. However, we will
deprecate the other spawn versions with CAF 2.0 and strongly encourage users to
write new code only using the best practices we have outlined.

## Traits for Typed Actors

When encountering compiler errors with `typed_actor`, it is often hard to
understand the compiler output due to the very long type names. To address this
we allow users to define traits for typed actors instead of passing the
signatures for the message handlers directly to the `typed_actor` template.

The trait is simply a user-defined class that contains a type alias
`signatures`, as shown in the following example:

```c++
struct cell_trait {
  using signatures = caf::type_list<
    caf::result<int32_t>(caf::get_atom),
    caf::result<void>(caf::put_atom, int32_t)
  >;
};

using cell_actor = caf::typed_actor<cell_trait>;
```

This quality-of-life change reduces the symbol name length in the compiler
output and makes it easier to understand the error messages. While we recommend
using this new approach for defining typed actors, the old way remains
supported.

## Improving the ABI Stability

While mostly an invisible change, we have also worked on improving the ABI
stability of CAF by better separating the implementation details from the public
API. With this release, we are adopting a new versioning policy: only major
releases may break the ABI (or API). We will continue to provide bug fixes and
new features in minor releases, but changes that affect the ABI will only be
made in major releases.

# Discontinuing CAFcademy

Last but not least, we have an important announcement to make regarding
CAFcademy.

With CAFcademy, we have provided a free online archive of learning materials for
CAF. Initially, we have also offered professional services around CAF on that
website, before we have re-organized our commercial offerings around CAF with
the launch of [Interance](https://www.interance.io) in 2022.

With the company operating all things CAF now, we have decided to discontinue
the CAFcademy brand, as it no longer serves a purpose. The articles from
CAFcademy have been completely overhauled to reflect the current best practices
and are available on
[https://www.interance.io/learning](https://www.interance.io/learning). The gems
articles from CAFcademy have been moved to the manual and are now part of the
official documentation. The line has always been blurry between the two and we
believe that this consolidation will make it easier for users to find the
information they need.

The CAFcademy website will be taken offline January 2025.
