---
layout: post
title:  "C++17, Breaking Changes, and Roadmap Update"
author: dominik
tags: API Roadmap
---

# Switching to C++17

The last time we have dropped support for a compiler was back in 2015 when we
removed support for GCC 4.7. While C++14 had several quality-of-life features,
we preferred stability over what C++14 had to offer. With C++17, the story is
different. Fold expressions and `if constexpr` alone are game changers for CAF.
That being said, we want to stay as conservative as possible when it comes to
our dependencies. So, we are not rushing towards latest GCC or Clang versions.

Compiler vendors have gotten a lot better at catching up to new language
versions and so did package maintainers and distributors. We did a survey on the
current landscape and found that all major platforms give their users easy
access to C++17 by now.

Microsoft has C++17 support in their Visual Studio Community Edition, Apple
ships very recent Clang versions with XCode, major BSD distributions such as
FreeBSD and OpenBSD come with C++17-ready versions of Clang, and, last but not
least, all major Linux distributors include at least one C++17-capable compiler
in their packages. On RHEL/CentOS,
[devtoolset](https://www.softwarecollections.org/en/scls/rhscl/devtoolset-7)
gives you access to GCC 7, SLES provides access to recent GCC versions and even
the oldest (still supported) LTS versions for Ubuntu and Debian have Clang 4.

This makes us confident that we are not leaving many (if any) users behind if we
raise our minimal required compiler version to one of:

- GCC ≥ 7
- Clang ≥ 4 (Apple Clang ≥ 9)
- Visual Studio ≥ 2019.

# Breaking Changes

With access to C++17, we are going to drop our replacement types such as
`caf::optional` and `caf::variant`. Rather than slowly fading in C++17 features
and deprecating individual parts one by one, we decided to make use of this
opportunity to reshape CAF as a whole. Since we are breaking the API by
switching to STL types and updating clumsy C++11 parts with more robust C++17
counterparts, we believe it is the right time to also apply lessons learned from
previous CAF iterations.

Ideally, CAF 0.18 breaks many things at once now, so that we can resume
incrementally evolving CAF after that point.

## Messaging

Over the course of the last years of developing CAF, we came to understand
`message` as a `variant`-like type able to hold any tuple type used for actor
messaging. The current design can play that role, but the implementation is a
highly flexible container that holds arbitrary data, offers views and even
enables composing messages in tree-like structures. This flexibility comes at a
cost. Obvious costs through virtual dispatch, but also runtime overhead for
matching message handler signatures to the content of incoming messages.
Further, because a message can essentially hold anything, CAF has to include a
lot of meta data in each message on the wire. This increases size overhead on
the wire but also makes deserializing messages costly.

Streamlining the messaging layer in CAF has the potential to improve performance
significantly. First prototypes for a new messaging layer speed up the matching
of messages to handlers as well as deserializing messages by a factor of 3. The
downside of that design is that users have to enumerate all allowed types in the
system. We did not reach full agreement whether we want to go down that route,
but it would also allow CAF to better target embedded systems, as the new
messaging layer would no longer require RTTI and stores less meta data.

## Serialization

Fold expressions and other C++17 features allow us to reimplement the
serializers in CAF more efficiently. Some of the optimizations are going to
require changes to the class hierarchy. We are most likely going to drop the
central `data_processor` and remove classes such as `stream_serializer` that CAF
no longer uses internally.

## OpenCL

A vendor-neutral API for GPGPU programming sure sounds great. Unfortunately,
OpenCL did not catch on in the way we had hoped. At this point, we can call
OpenCL dead and gone. There is only legacy support available and recent versions
of the standard were never implemented in the first place. Consequently, we are
going to drop `caf_opencl`.

## Networking

Unrelated to CAF 0.18, we are working on a new networking layer in CAF. While we
generally like the actor-based Broker API of `caf_io`, the module struggles to
integrate exchangeable transport layers. Ideally, `caf_openssl` would not have
to come as separate module.

With [`caf_net`](https://github.com/actor-framework/incubator), we are currently
working on a clean slate design that gives more flexibility in deployment while
at the same time improving the user experience. We based the new API on URIs,
which makes is easy to select a transport via URI scheme. We are also working on
name-based access to remote actors that replaces the somewhat quirky
`publish`/`remote_actor` function pair of `caf_io`.

However, `caf_net` is not going to replace `caf_io` overnight. Once the
`caf_net` module reached a stable version, we will move it into the main
repository where it eventually replaces `caf_io`.
