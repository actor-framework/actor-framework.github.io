---
layout: post
title: "Spotlight: Bro"
author: dominik
tags: Spotlight Bro
---

For our second article in the spotlight series, we want to higlight the popular
open source network monitor [Bro](https://www.bro.org) and talk to the core
development team in Berkeley, California.

# Background: Bro(ker)

<center>
  <img src="{{ site.url }}/static/img/blog/bro.png" alt="Official Bro Logo">
</center>

<br>

Bro is an open source network analysis framework, well grounded in 15 years of
research. While focusing on network security monitoring, Bro provides a
comprehensive platform for more general network traffic analysis as well. Bro's
user community includes major universities, research labs, supercomputing
centers, open-science communities, and has an established foothold in the
industry. In addition to organize user conferences and providing online
resources for learning and using Bro, the creators of Bro also offer
enterprise-level commercial support.

With [version 2.4](http://blog.bro.org/2015/05/bro-24-beta.html), the Bro
development team integrated a new communication library based on CAF called
[Broker](https://github.com/bro/broker) (not to be confused with brokers in
CAF).  Currently, this new communication layer is shipped as beta release and
needs to be enabled explicitly during build. Broker will become a mandatory
dependency in future Bro versions and replace the current communication and
serialization system.

In this interview, we have the pleasure to talk to the core development and
research team primarily located at the [International Computer Science
Institute (ICSI)](http://www.icsi.berkeley.edu) in Berkeley, California.

# The Interview

__CAF Team__: Dear Bro team, thank you for giving us the opportunity for
this interview. The original paper describing Bro was released in 1999.
This is a very long time for a software code base and clearly speaks for
its quality. What is your secret?

__Bro Team__: When Vern Paxson created the initial version Bro, he designed the
system out of practical experience and---as we can say in hindsight---came up
with the right abstractions in the system architecture, which at its core hasn't
changed since.

We also carefully try to avoid feature creep and only add new functionality when
finding convincing use cases, which often involves discussion. While this
sometimes comes at the of cost development speed, it often helps us to make
strategically viable decisions in the long term.

__CAF Team__: Bro is a security-critical software and network operators
around the world rely on it. Changing a key component like the
communication system is not something that you do without having a good
reason. What was your motivation to start developing a new communication
backend from scratch?

__Bro Team__: To keep monitoring the ever-growing network uplinks at line rate
today (10, 40, or even 100 Gbps), the traffic needs to be distributed across
several physical systems. Logs are sent back to a single system, where they need
to be quickly and reliably written out. In addition, certain types of analysis
are distributed across all of the nodes (e.g. scan detection) and stress the
communications framework.

Our existing communication framework starts to show its age: there exist race
conditions with synchronization of high-level state, it lacks a persistence
component, the pub/sub data flow cannot be controlled in a fine-grained way, the
implementation is complex and inefficient, and we have to maintain two
independent implementations: there exists a separate C library (Broccoli) to
allow external applications to interact with Bro.

__CAF Team__: What goals did you set for the new communication system? What do
you want to achieve with the system in the long run?

__Bro Team__: Ultimately we want to "crack open" the boundaries of what
constitutes Bro and who can communicate with it. We also want to consolidate our
existing implementation into a single library, which both Bro and external
applications can use. In the past, Bro spawned a separate child process for its
communication. Now we're just linking against a shared library---Broker---which
provides a uniform API to communication in the Bro ecosystem.

We want to create a platform where it's easy to communicate with Bro instances,
reconfigure them on the fly, and change their state not just from within Bro,
but from anywhere. Moreover, we aim to provide a more flexible pub/sub event
communication infrastructure.

__CAF Team__: Many challenges in software development arise in the
implementation step and are not obvious in the initial design phase.
What were the core issues and challenges you faced when implementing
Broker and how did CAF contribute to your solution?

__Bro Team__: From a high level, ripping out a communication infrastructure and
replacing it with a new one, *without* users even noticing, is a big challenge.
This means we have to be able to incrementally deploy Broker. We already shipped
Broker with Bro 2.4, but it's not yet enabled by default. However, our master
branch already relies on Broker, which means that 2.5 will come with Broker
enabled by default after we get enough feedback from our users and putting out
release candidates. The transition to C++11 so far caused most of the attention.

From the implementation side, we wanted to converge to a tight, minimal API that
proves powerful enough to cover our use cases. CAF gives us the flexibility we
need to hide a powerful asynchronous backend behind a blocking C API, but also
expose the asynchrony at at Broker's C++ API. Then there's the messaging aspect:
Broker is essentially a distributed pub/sub engine which naturally maps to the
abstractions CAF provides.

One of our biggest challenges involves bringing together asynchronous lookups
and type-erasure in the Bro's statically typed scripting language. For example,
when asking a remote data store for a value, the return type is not known a
priori. Fortunately, we already have proposal for adding pattern matching (note
to the security community: not regular expressions) for type-safe message
access.

__CAF Team__: Did you consider other frameworks? If yes, what convinced
you to use CAF instead?

__Bro Team__: We looked at other, more low-level messaging libraries. In
particular, we compared CAF to nanomsg. With CAF we found that we could express
the same idea in significantly less code because it comes with everything we
need of the box. CAF simply doesn't require any boilerplate and enabled us to
focus on the problem domain, as opposed to grappling with the implementation
details of messaging framework. After all, avoiding complexity and being able to
reason about both code and protocol was the primary motivator for rewriting our
communication layer.

The type-safe abstractions CAF provides on top of bare messaging made a huge
difference in productivity. One of our team members summarized it as "[..] it
didn't feel like I was writing code that attempts to *implement* a protocol as
it felt like writing code that *is* the protocol."

We also found that nanomsg's communication patterns, which represent one of its
key selling points, can be easily implemented in a few lines of code with CAF.
Orthogonal to that, CAF comes with a nice failure semantics (monitors & links)
which we haven't found in other frameworks.

Finally, if you didn't release CAF under a BSD (or equivalently permissive)
license, we couldn't even have considered it in the first place.

__CAF Team__: How were the reactions to CAF in your team so far?

__Bro Team__: Internally we had one CAF fan already who brought the actor model
to the attention of the team. After our comparative evaluation, we were all
excited to see such a clear winner and looking forward to seeing CAF perform in
more than just toy examples.

__CAF Team__: Can you tell us about your performance and scaling
requirements? Did CAF meet your expectations?

__Bro Team__: Broker enables us to reach a new horizon in terms of scaling. Bro
is one of the unique open system capable of monitoring 100 Gbps---a workload
which only a cluster of machines can stem. The worker nodes must send the logs
somewhere via Broker, while the messaging layer must not induce a high overhead
so that the nodes can focus on their traffic analysis.

In addition to the standard cluster for monitoring a single link, we have an
ongoing "deep cluster" project where multiple sensor nodes deep in the network
augment the traditional boarder-gateway monitoring. This entails deploying
hundreds and thousands of nodes at global scale.

Finally, Bro can instruct open-flow controllers to reconfigure network hardware,
e.g., to insert blocking rules or shunt flows. These use cases have tight
latency requirements, with Broker must meet.

We are still in the process of evaluating performance; things look good so far
but we expect to do more tuning as our users start to rely on it for operations.

__CAF Team__: The official announcement of Broker was at [BroCon
2015](https://www.bro.org/community/brocon2015.html). How did your
community react to your plans for the communication system? Did they see
the benefits immediately or did they object to changing critical
components and introducing CAF as a new long-term dependency?

__Bro Team__: Our users liked that Broker enables new ways of communicating with
Bro in the future. We're already receiving first questions about Broker on the
mailing list.

Fortunately, our community trusts us developers that we make the right strategic
decisions about Bro's future. So we haven't heard any critical voices about
introducing CAF as new dependency. In fact, moving to C++11 will allows us to
modernize Bro's code base itself.

__CAF Team__: Final question: if you were to decide what the next
feature of CAF would be, what would you have in mind?

__Bro Team__: In the near term, we want to migrate to typed actors so that we
can get a compile-time proof that our messaging protocol implements the spec
correctly. But CAF can do that already. In the short term, we want to use the
direct connection optimization for when distant siblings in a tree hierarchy
communicate. Further down the road, we're excited about the planned
introspective features, where CAF can report utilization of CPU I/O, which we
would like to use a basis for decisions to adapt our topology dynamically at
runtime. CAF's stateful and migratable actors seem to lay the foundation for
this capability.

__CAF Team__: Thank you very much for this interview and we wish you all
the great success for the next 15 years of Bro!


Special thanks go to Matthias Vallentin for introducing the teams to each
other. Matthias is a member of the Bro team as well as a long-time contributor
to CAF.

# Links

* __Bro Project__: <https://www.bro.org>
* __Bro Github__: <https://github.com/bro>
* __Bro Twitter__: [@Bro_IDS](https://twitter.com/Bro_IDS)
* __Broker__: <https://github.com/bro/broker>
