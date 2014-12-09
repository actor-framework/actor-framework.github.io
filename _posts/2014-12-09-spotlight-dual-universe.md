---
layout: post
title: "Spotlight: Dual Universe"
author: dominik
tags: Spotlight DualUniverse
---

In our new spotlight series, we present software projects that are based on
CAF. The first project we want to highlight is the Massive Multiplayer Online
(MMO) game *Dual Universe*. We talked to Jean-Christophe Baillie, founder and
president of Novaquark, the company behind the game.

# Background: Dual Universe

[<img class="centered-image" src="static/img/dualuniverse.jpg" width="640" height="360" alt="Official Dual Universe Artwort">](static/img/dualuniverse.jpg)

Dual Universe is a science fiction, multi-planet world where "players are free
to invent their collective destiny: civilizations will rise and fall,
player-driven events will shape the course of history". The ambitious goal is
to implement a persistent single-shard universe consisting of thousands of
procedurally generated planets. Based on the Unreal Engine 4, Dual Universe
strives to deliver beautiful and immersive environments. It will also be
compatible with Oculus Rift. To see some of the impressive artwork and read
more about the game, visit [the official homepage](http://www.dualthegame.com).
The first alpha release will hopefully be available in mid 2015.

Scaling up a software infrastructure to simultaneously deal with millions of
active players is a real challenge. To build this scalable infrastructure, the
development team behind Dual Universe has selected CAF.

# The Interview

__CAF Team__: Dear Jean-Christophe, thank you for giving us the opportunity for
this interview. You are the founder and president of Novaquark, but given the
relatively small team size of indie studios, we can imagine you are also deeply
involved in the development process.

__Jean-Christophe__: The team is currently 18 people, with 7 developers. I
wrote a first prototype of the game two years ago, as well as the server
architecture and scalability algorithms. Because our ambition has grown a lot
since the beginning of the project, most of the original prototype has been
rewritten, with the exception of the server architecture, which is still in
operation.  I currently supervise the technical developments, as well as the
game design in general.

__CAF Team__: Building an MMO is a very ambitious goal for an indie studio.
What is the history behind and what are your plans for the future?

__Jean-Christophe__: Yes, developing a MMO as an indie studio can sound a bit
crazy! The important thing to see is that the design of the game is based on
what is called "emergent gameplay": the players are going to create their
world, their stories, and their destiny. Unlike other MMOs, we don't create
tons of content like cities or quests, but we give the players a sandbox to
create their own. This is making the project more feasible as we are more a
tech company than a content company. 

As for the origin of the project: it's a very old gamer dream, something I've
been thinking about for at least 6 or 7 years now. Dual Universe is a
single-shard sandbox MMO taking place on hundreds of planets, where players are
free to create their own economic or political systems, gathered in
organizations and explore the procedurally generated world for resources and
opportunities.  Quite uniquely, they also are able to modify the voxel-based
world: create cities, various scriptable constructs like vehicles, space ships
or gigantic orbital stations. All players share the same "single shard"
universe : there is only one persistent reality, giving birth to emergent
organizations and player specialization around activities like building,
politics, security, piracy, space colonization and exploration, logistics,
harvesting, manufacturing or industry production.

We hope that we will see civilizations rise and fall, empires and alliances
unite or compete, in a totally open world where everything will be possible.

__CAF Team__: How did you learn about CAF and what convinced you that it is the
right tool for building your infrastructure? After all, this is a most critical
business component. Did you try other frameworks before?

__Jean-Christophe__: Originally, I was interested in scalable design patterns
to create a robust and distributed server architecture. I knew about Erlang and
got interested in the actor programming model. Erlang sounded great, but we
finally decided to use Scala and Akka to build a first prototype of the
architecture (the Java proximity sounded like a good way to stay connected to a
rich ecosystem). It was very hard however to find good Scala developers and we
quickly realized that finding a way to get the job done in C++ would make
things much easier in terms of recruiting. But C++ is a very dangerous language
to use for a server: it's not as safe, and does not provide native scalability
and actor-based features like Scala/Akka does. We decided to take advantage of
modern C++11 features and design patterns to try to avoid the typical memory
management problems of C++ (plus tons of other issues), and I started to look
for a good implementation of the actor model in C++. That's when I found
libcppa, the old name of CAF, and started to test it. CAF is very close in
philosophy to Erlang/Scala/Akka, and is based on sound C++11 programming, so we
fell in love with it pretty quickly!

__CAF Team__: What did you experience with CAF? How fast could you get
productive?  Did you have a long evaluation phase?

__Jean-Christophe__: CAF was very easy to use, with well documented examples. I
think my familiarity with the actor model was probably helping, I suppose the
particular way of doing things might be more difficult to grasp if you are
totally new to this approach. 

__CAF Team__: What technologies do you use beside CAF and what are your
experiences in integrating CAF with other tools and frameworks?

__Jean-Christophe__: We use an external middleware to handle network
communications with processes not running CAF (mainly, the game client); we
also deal with a few standard databases and external data stores. All of them
behave with their own logic: synchronous vs. asynchronous, single or
multi-threaded. We developed wrappers for those to keep the functional logic as
much as possible in CAF actors. Each tool is specific, but we mainly use
`caf::broker`s, and anonymous sends (`anon_send`) from the threads not managed
by CAF to exchange data with the actors.

__CAF Team__: Can you tell us about your scaling requirements? What is your
experience (and expectation) regarding performance?

__Jean-Christophe__: As I said, the novelty we try to bring to the market is
the idea of a continuous single-shard universe. This means that we must be able
to balance the computing load dynamically, whereas current MMO games balance it
statically, depending on the player position in the game. 

CAF allows us to avoid dealing with low-level synchronization issues (data
races, deadlocks, costly synchronization primitives) while enabling the logical
architecture of our code. Furthermore, we leverage CAF network transparency to
let actors communicate directly, whether in the same process or across our
datacenter. This is a huge step towards fluid scalability, since it eases
considerably the possibility to throw more hardware and increase the processing
power linearly.

So far, we are satisfied with the performance, and we plan to build large-scale
stress tests to put even more pressure to our architecture.

__CAF Team__: You have been using CAF for quite a while and started using it
when it was still named libcppa. What was the first version you have used and
what was your experience with the progress of CAF during that time?

__Jean-Christophe__: I think the first version was libcppa 0.8. It was already
fairly mature and reliable. We discovered small issues that were quickly solved
in the next iterations and globally, I would say that the reactivity of the
team has been very good.

__CAF Team__: Final question: if you were to decide what the next feature of
CAF would be, what would you have in mind?

__Jean-Christophe__: I can think of a few, some of which are probably a bit
below the scope of CAF! 

- `atom()`s with more than 10 characters!
- when `announce()`ing a custom type, `std::container<type>` should be announced
  too; or at least it would be nice to detect at compile-time that it cannot be
  sent as a `message`
- the ability to migrate actors between processes - though we are not sure if it
  would make sense - to further enhance dynamic scalability
- an easy way to view internal metrics, such as: number of messages handled per
  second, hotspots in the application, number of message waiting in a queue,
  etc.

__CAF Team__: Thank you very much for this interview and we wish you and your
team all the great success!

# Links

* __Twitter__: [/dualthegame](https://twitter.com/dualthegame)
* __Facebook__: [/dualuniverse](https://www.facebook.com/dualuniverse)
* __Linkedin__: [/company/novaquark](https://www.linkedin.com/company/novaquark)
