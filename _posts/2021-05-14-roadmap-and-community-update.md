---
layout: post
title:  "Roadmap and Community Update"
author: dominik
tags: API Roadmap Community
---

# CAF 0.18 Retrospective

With CAF 0.18, we increased the robustness and performance of the messaging
layer and re-designed the type inspection API. These changes required breaking
the API and after helping several users migrate to the new framework version, we
know that the migration is not a trivial one. However, the experience reports
that reached us confirmed that this was indeed the right step for CAF. In terms
of performance, but also because the changes enabled new exciting features for
CAF such as the
[JSON serialization](https://www.cafcademy.com/gems/json-serialization).

CAF 0.18 also added the new
[Metrics API](https://actor-framework.readthedocs.io/en/stable/Metrics.html)
that enables users to monitor their CAF applications with
[Prometheus](https://prometheus.io).

Looking back after five months (8 when counting from RC-1), we think the 0.18
release turned out the way he had hoped and gives us a solid foundation to work
from.

# Upcoming changes with CAF 0.19

With the next version of CAF, we are going to complete the transition to C++17
and tie up some loose ends of 0.18:

- With access to C++17, time has come to fade out CAF backports of standard
  library features such as `optional`, `variant`, and `string_view`.
  Consequently, we will deprecate the CAF types in favor of their `std`
  equivalent.
- With CAF 0.18, we have changed the internal representation for `typed_actor`
  to simply use C++ function reference syntax, e.g.,
  `typed_actor<result<int32_t>(get_atom)>`. This rendered the metaprogramming
  utilities `replies_to` and `reacts_to` obsolete and CAF 0.19 is going to
  deprecate them.
- After seeing the remote spawn feature used in production by some users with
  generally positive feedback, we feel confident to remove the `experimental`
  tag for this API.

# Planned Features

Aside from the maintenance-focused changes for 0.19, we are also working on a
couple of new features. At this point, we cannot tell whether any of these find
their way into CAF 0.19 or will have to wait for a later release.

## Revisiting the Streaming API

We are working on a new streaming API for CAF that borrows many concepts from
[Reactive Streams](https://www.reactive-streams.org). The new streaming API is
going to replace the previous experimental API that exists since version 0.16.

We decided to give the streaming API a complete overhaul since the previous
version only works well for simple pipelines but struggles to express more
complex work flows. In particular, the new API is going to make it much easier
to define data flows across actor boundaries to make interactions with non-actor
parts of the system more robust and straightforward.

## Better Unit Testing

The unit testing API in CAF is also in for a complete overhaul. We found the
tight integration with the unit testing framework proved invaluable for CAF for
testing individual actors as well as complex interactions between actors.
However, the original design dates back to 2015 and shows its age. Over the
years, the API acquired multiple layers of features scattered across multiple
headers to interface with increasing capabilities of CAF such as the
deterministic scheduling and the test clock for manually advancing time.

Aside from API inconsistencies, the unit testing framework comes with a default
`main` implementation that is flat out broken since CAF 0.18 because it does not
initialize meta objects for the type ID blocks.

## New Networking Capabilities

When speaking of outdated designs, we also must talk about `caf_io`. The idea of
*broker actors* that manage connections seemed a natural fit for a framework
like CAF. However, `caf_io` has a hard time scaling beyond a single thread since
each broker may hold multiple sockets and introducing concurrency would
introduce heavy need for synchronization. Moreover, `caf_io` makes it easy to
interface with actors but hard to compose protocol stacks or to re-use existing
brokers with different communication backends. Ideally, CAF would not need a
complete new module such as `caf_openssl` only to change the transport protocol.

With `caf_net`, we have started from scratch to implement the next generation of
networking for CAF. The new module shifts the focus to making composing protocol
stacks easy. We have put a lot of effort into the early design phases but it
probably is going to take quite  a bit of time before `caf_net` is ready to
fully replace `caf_io`. If you are curious about the new API, head over to our
[incubator](https://github.com/actor-framework/incubator). Some parts of
`caf_net` are already stable enough to encourage people to try it out. For
example, the new
[WebSocket](https://github.com/actor-framework/incubator/tree/master/examples/net)
protocol bindings.

In any case, `caf_net` is not going to replace `caf_io` over night. When the new
module reaches a stable release, we will move it to the main repository. Only
after collecting enough experience with the new module are we going to deprecate
(and eventually remove) `caf_io`.

# Road to Version 1.0

CAF has proven itself in production in multiple industries over the years. It is
about time we release a version 1.0 to reflect this. The purpose of CAF 0.19 is
primarily to "wrap up" what 0.18 started and to allow us to release version 1.0
without obsolete legacy types.

# Beyond the Code

As many of you probably noticed, we launched
[CAFcademy](https://www.cafcademy.com) earlier this year. The website offers
commercial services to CAF users and provides free online learning resources for
anyone interested in the framework. It currently contains a first couple of
articles that teach what we have identified as best practices over the years. We
have more articles planned, so stay tuned!

Of course, feedback on the current content is most welcome, e.g., by dropping us
an email under [feedback@cafcademy.com](mailto:feedback@cafcademy.com) or by
reaching out on [Twitter](https://twitter.com/actor_framework)! Are we covering
the right topics? Are the articles helpful and easy to follow? Are the provided
code snippets useful and comprehensible?

CAF started as a research prototype and the framework would not have reached
this state without the ongoing support and dedication from the
[iNET research group](https://www.inet.haw-hamburg.de). Of course there are
constraints to what services we can offer to professional users as researchers.
This is why we are in the process of building a sustainable business model.

Since early 2020, I fully shifted my professional focus to CAF. This includes
improving the framework itself as well as helping companies build and maintain
CAF-based software. We are open source enthusiasts and you can rely on CAF to
stay free and BSD-licensed!

On the business side, our top priority right now is building a reliable income
source for funding continued development of CAF and growing our team. Support
agreements are the most common way for open source projects to build a stream of
revenue. We are of course open to explore other ideas as well. If your company
is using CAF (or is considering it), I would love to hear from you!

In fact, I would love to hear from you even if you are using CAF "only" for your
spare-time projects! The more feedback we get, the better we can improve the
framework and prioritize what matters most to our users.
