---
layout: post
title:  "A Glimpse at the Future of CAF"
author: dominik
tags: API
---

Sometimes, a design outlives its usefulness. CAF started with its first commit
on March 4, 2011. Its mission statement? Provide a lightweight, domain-specifc
language (DSL) for actors in C++. The user-facing API should be as minimalistic
as possible by hiding internal state of the actor system. Components such as
the scheduler or the middleman are lazily initialized singletons. This design
works great as long as applications do not wish to configure individual
components of the actor system. With a growing userbase and more fields of
applications, it becomes clear that hiding as much state of the system as
possible puts obstacles in the way of many users.

After much consideration, we decided to start over with a new API design in
0.15. The next release of CAF embodies our vision for a 1.0 release and
addresses all the feedback we collected over the years. The new anchor of CAF
applications is `actor_system`, which replaces the previous singleton-based
design. Users are in full control over the setup phase of actor systems and
most of the system can now be configured via `.ini` files. We also replaced
misleading function names like `sync_send` with more intuitive names and
improved the garbage collection based on reference counting. Without further
ado, this is how the hello world example will look like in 0.15:

```cpp
behavior mirror(event_based_actor* self) {
  // return the (initial) actor behavior
  return {
    // a handler for messages containing a single string
    // that replies with a string
    [=](const string& what) -> string {
      // prints "Hello World!" via aout (thread-safe cout wrapper)
      aout(self) << what << endl;
      // reply "!dlroW olleH"
      return string(what.rbegin(), what.rend());
    }
  };
}

void hello_world(event_based_actor* self, const actor& buddy) {
  // send "Hello World!" to our buddy ...
  self->request(buddy, std::chrono::seconds(10), "Hello World!").then(
    // ... wait up to 10s for a response ...
    [=](const string& what) {
      // ... and print it
      aout(self) << what << endl;
    }
  );
}

int main() {
  // our CAF environment
  actor_system system;
  // create a new actor that calls 'mirror()'
  auto mirror_actor = system.spawn(mirror);
  // create another actor that calls 'hello_world(mirror_actor)';
  system.spawn(hello_world, mirror_actor);
  // system will wait until both actors are destroyed before leaving main
}
```

For comparison, this is the same code in 0.14:

```cpp
behavior mirror(event_based_actor* self) {
  // return the (initial) actor behavior
  return {
    // a handler for messages containing a single string
    // that replies with a string
    [=](const string& what) -> string {
      // prints "Hello World!" via aout (thread-safe cout wrapper)
      aout(self) << what << endl;
      // terminates this actor ('become' otherwise loops forever)
      self->quit();
      // reply "!dlroW olleH"
      return string(what.rbegin(), what.rend());
    }
  };
}

void hello_world(event_based_actor* self, const actor& buddy) {
  // send "Hello World!" to our buddy ...
  self->sync_send(buddy, "Hello World!").then(
    // ... wait for a response ...
    [=](const string& what) {
      // ... and print it
      aout(self) << what << endl;
    }
  );
}

int main() {
  // create a new actor that calls 'mirror()'
  auto mirror_actor = spawn(mirror);
  // create another actor that calls 'hello_world(mirror_actor)';
  spawn(hello_world, mirror_actor);
  // wait until all other actors we have spawned are done
  await_all_actors_done();
  // run cleanup code before exiting main
  shutdown();
}
```

Most notably, the `main` function no longer calls `await_all_actors_done()` and
`shutdown()`. Nor needs the `mirror` actor to call `quit` explictly. Once
the `system` variable goes out of scope, it will wait for all remaining actors
before cleaning up. The `mirror` is destroyed implicitly once there is no more
reference to it. Finally, `sync_send` has been renamed to `request` and now
requires a timeout.

Without going into too much detail, dynamically and statically typed actors get
a symmetric API. We also provide new ways to compose actors and the behavior of
actors. If you are interested in the current state of development, check out
the `topic/actor-system` branch. It is in development since November 2015,
currently contains 200 commits, and will get merged into `develop` soon.

Since version 0.15 has many API changes, we will release at least one
pre-release version for collecting user feedback and finalizing the API. The
first pre-release is scheduled for June and will include a complete overhaul of
the user manual. It was not an easy decision to break the API on so many
layers. However, the changes are the result of all the feedback we received
over the years as well as our own experiences with using CAF on a daily base.
This makes version 0.15 the most important milestone towards a stable 1.0
release.
