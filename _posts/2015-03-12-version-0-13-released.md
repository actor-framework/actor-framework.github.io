---
layout: post
title:  "Version 0.13 released!"
author: dominik
tags: API GitHub
---

Version 0.13 of CAF has just been released.  This release contains mostly
improvements to core components, but also deprecates some parts of the API and
removes previously deprecated parts.

# No `cppa` Headers

The `cppa` headers were deprecated with 0.9 and are now finally removed. In
case you were using the old `cppa::options_description` API, you can migrate to
the new API based on `message::extract` (see Section 18.4 of
[the Manual](http://actor-framework.org/manual)).

# IPv6 Support

You can now use IPv6 addresses for `remote_actor`.

# Deprecate `last_dequeued` and `last_sender`

Version 0.13 slightly changes the semantics of `last_dequeued` and
`last_sender`. Both functions will now cause undefined behavior (dereferencing
a nullptr) instead of returning dummy values when accessed from outside a
callback or after forwarding the current message. Besides, the function names
were not a good choice in the first place, since "last" implies accessing data
received in the past. As a result, both functions are now deprecated. Their
replacements are named `current_message` and `current_sender`.

# Pattern Matching Revised

The pattern matching engine no longer accesses the RTTI system of CAF. This
means that you no longer have to announce your messages as long as you use only
the core component of CAF for in-process messaging and *not* use `to_string`.
The main reason for this change was to reduce the amount of template
instantiations and to make CAF more debugger-friendly. For example, consider
the following simple CAF program:

<script src="https://gist.github.com/Neverlord/a5efdca15300adc421db.js"></script>

Prior to 0.13, this produces the following three stack traces (using `c++filt`
and removing unwanted parts such as binary name):

<script src="https://gist.github.com/Neverlord/aaf212e49e3ed7ff46ab.js"></script>

If you ever ran into a message handler with your debugger of choice, you had to
filter through *a lot* of templates and CAF internals.

Starting with 0.13 (and applying the same transformations), you will get these
stack traces instead:

<script src="https://gist.github.com/Neverlord/f46235ccd65b9471175e.js"></script>

There are still a few templates left, but overall it is much easier to
understand what is going on. Whenever you pass a lambda without any prefix, you
will get a `trivial_match_case`. Using the "`on(...) >> ...`" notation results
in an `advanced_match_case`. Lastly, "`others() >> ...`" will produce a
`catch_all_match_case`. The function `apply_args` takes a function or function
object, a (compile-time) list of indexes, and a tuple. It then simply gets all
values using the given indexes from the tuple and calls the function object.
The `lfinvoker` is a simple helper that returns `unit` whenever your message
handler would return `void`. Of course you will see a bit more CAF internals
when using event-based actors, but the old `match_case` madness is gone for
good.
