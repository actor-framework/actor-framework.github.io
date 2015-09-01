---
layout: post
title: "Spotlight: Bro"
author: dominik
tags: Spotlight Bro
---

For our second article in the spotlight series, we want to higlight the popular
open source network monitor Bro and talk to the core development in Berkeley,
Colorado. Special thanks go to Matthias Vallentin for introducing the teams to
each other. Matthias is a member of the Bro team as well as a long-time
contributor to CAF.

# Background: Bro(ker)

<center>
  <img src="{{ site.url }}/static/img/bro.png" alt="Official Bro Logo">
</center>

<br>

Bro is an open source network analysis framework, well grounded in 15 years of
research. While focusing on network security monitoring, Bro provides a
comprehensive platform for more general network traffic analysis as well. Bro's
user community includes major universities, research labs, supercomputing
centers, and open-science communities. In addition to organize user conferences
and providing online resources for learning and using Bro, the creators of Bro
also offer enterprise-level commercial support.

To read more about Bro's adaptable, efficient, and flexible forensics
infrastructure, visit [the official homepage](https://www.bro.org).

With [version 2.4](http://blog.bro.org/2015/05/bro-24-beta.html), the Bro
development team integrated a new communication library based on CAF called
*Broker* (not to be confused with brokers in CAF).  Currently, this new
communication layer is shipped as beta release and needs to be enabled
explicitly during build. Broker will become a mandatory dependency in future
Bro versions and replace the current communication and serialization system.

In this interview, we have the pleasure to talk to the core development and
research team at the International Computer Science Institute in Berkeley, CA.

# The Interview

__CAF Team__: Dear Bro team, thank you for giving us the opportunity for this
interview. The original paper describing Bro was released in 1999. This is a
very long time for a software code base and clearly speaks for its quality.
What is your secret?

__Bro Team__: ...

__CAF Team__: Bro is a very sensible piece of software, as network
administrators around the world rely on it. Changing a key component like the
communication system is something that you do not do without having a good
reason. What was your motivation to start developing a new communication
backend from scratch?

__Bro Team__: ...

__CAF Team__: After settings goals for the new systems, when did you decide
that CAF was the right tool to achieve those? Did you try other frameworks
before?

__Bro Team__: ...

__CAF Team__: How were the reactions to CAF in your team so far?

__Bro Team__: ...

__CAF Team__: Can you tell us about your performance and scaling requirements?
Did CAF meet your expectations?

__Bro Team__: ...

__CAF Team__: The official announcement of Broker was at BroCon 2015. How did
your community react to your plans for the communication system? Did they
see the benefits immediately or did they object to changing critical components
and introducing CAF as a new long-term dependency?

__Bro Team__: ...

__CAF Team__: Final question: if you were to decide what the next feature of
CAF would be, what would you have in mind?

__Bro Team__: ...

__CAF Team__: Thank you very much for this interview and we wish you all the
great success for the next 15 years of Bro!

# Links

* __Twitter__: [/Bro_IDS](https://twitter.com/Bro_IDS)
* __Facebook__: [/thebroplatform](https://www.facebook.com/thebroplatform)
