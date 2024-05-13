---
layout: post
title:  "Simplifying Send: the new Mail API"
author: dominik
tags: API
---

# Background

In CAF, we have multiple ways to send messages today: `send` for sending an
asynchronous message, `scheduled_send` for sending a message at a specific time,
and `delayed_send` for sending a message after some delay. For request-response
communication, we have `request`. However, there is no function for sending a
request after some delay. Users can emulate that by using `run_scheduled` and
send the request message in the callback. Further, the `run_scheduled` functions
also have a weak version, which `scheduled_send` has not. Actors can also
delegate messages to other actors using `delegate`, adding even more functions
to the mix. All these functions are fundamentally about sending messages, but
there is little coherence in the API design. These functions were introduced at
different times in the development process of CAF and for different use cases,
leading to a fragmented API. This makes the API both hard to navigate for users
and hard to maintain for our team.

Sending messages is a fundamental operation in CAF. Hence, changing this API
requires careful consideration, as it affects many users. After collecting
feedback from our community and discussing the topic in our team, we decided
that while introducing a new API will put burden on existing users that have to
migrate their code, the benefits of a new API that simplifies both using and
extending CAF outweigh the costs in the long run.

With CAF 1.0, we introduce a new mail API that simplifies sending messages. The
new API is more flexible, allows for more control over the message sending
process, and will allow us to change implementation details in the future
without breaking user code.

After upgrading to CAF 1.0, users should be prepared to see deprecation warnings
for the old API. Hence, we want to use this blog post to introduce the new API
early on and give some examples to guide users through the transition.

As we will see in the examples, the transition is straightforward. In fact, the
transition can be automated in most cases and we will provide a tool to assist
developers with the migration to the new API as a service for our customers with
a support contract.

# The new API in a Nutshell

With CAF 1.0, every send operation starts with `mail(elements...)`, whereas
`elements...` is the content of the message. Afterwards, users can optionally
set a priority and a delay (or timeout) before calling either `send` or
`request`.

Here are some examples to illustrate the difference between the old and the new
API:

```cpp
// (1) old
self->send(worker, 1, 2, 3);

// (1) new
self->mail(1, 2, 3).send(worker);

// (2) old
self->send<message_priority::high>(worker, 1, 2, 3);

// (2) new
self->mail(1, 2, 3).urgent().send(worker);

// (3) old
self->delayed_send(worker, 1s, 1, 2, 3);

// (3) new
self->mail(1, 2, 3).delay(1s).send(worker);

/// (4) old
self->request(worker, 2s, 1, 2, 3).then...

/// (4) new
self->mail(1, 2, 3).request(worker, 2s).then...
```

The new API is not just replacing the old one, though. Here are some examples
that have no direct equivalent in the old API:

```cpp
// delay a request
self->mail(foo).delay(1s).request(worker, 2s).then...

// delay a request and keep a weak reference to the receiver until sending
self->mail(foo).delay(1s).request(worker, 2s, weak_ref).then...

// delay a request and keep only weak references until sending
self->mail(foo).delay(1s).request(worker, 2s, weak_ref, weak_self_ref).then...
```

For sending messages anonymously, users can use `anon_mail` to replace the old
`anon_send` function.

# Blocking Receive

When using scoped actors, the new API also provides the blocking `receive`
function that the old API offers. However, there is a new version of `receive`
that takes no arguments and returns an `expected` for the response. Here are
some more examples:

```cpp
// (1) old
self->request(worker, 1s, 1, 2, 3).receive(
  [](int result) { /* ... */ },
  [](const error&) { /* ... */ });

// (1) new
self->mail(1, 2, 3).request(worker, 1s).receive(
  [](int result) { /* ... */ },
  [](const error&) { /* ... */ });

// (2) old
auto f = make_function_view(calculator); // typed
auto res = f(1, 2);

// (2) new
auto res = self->mail(1, 2).request(calculator, 1s).receive();      // typed
auto res = self->mail(1, 2).request(calculator, 1s).receive<int>(); // untyped
```

Since the use case of `function_view` is covered by the new API, we will
deprecate `function_view` in CAF 1.0 as well.

# Conclusion

While breaking changes are always a tough decision, we believe that the new API
will simplify the use of CAF, makes it easier to teach, and will allow us to
improve the implementation in the future while keeping the API stable. We are
very grateful for the feedback we received from our community and hope that the
new API will result in a better experience for all users.
