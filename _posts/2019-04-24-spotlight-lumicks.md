For this spotlight, we dive into the world of molecular research and talk to
the software team of the highly ambitious startup Lumicks at their Amsterdam
HQ.

# Background: Lumicks

<img class="centered-image" src="{{ site.url }}/static/img/lumicks.png"
alt="Official Lumicks Logo">

Lumicks manufactures dynamic single-molecule and cell avidity analysis
equipment. Their breakthrough technologies enable visualization of molecular
interactions and acoustic manipulation of biomolecules. With their products,
Lumicks helps scientists to understand life to the smallest detail, which is
critical for cancer research and drug development.

Founded in 2014, Lumicks quickly became a success story. Today, research sites
all around the world have adopted Lumicks technology: from UC Berkeley at the
U.S. west coast all the way to ShanghaiTech University in far east China.

# The Interview

__CAF Team__: Dear Lumicks team, thank you for taking the time speaking to us!
Lumicks works hard to deliver unique insights into biological processes at a
molecular level. A [recent article in The
Scientist](https://www.the-scientist.com/modus-operandi/dancing-cells-65127/amp)
speaks about using sound waves to capture cells, which is a process you use for
some of your products, while other products mention optical tweezers. A bit
exaggerated: How can lasers and music help us to cure cancer? Without going
into too much detail, how would you explain what your products do for people
like us that don't have degrees in molecular biology?

__Lumicks Team__: The key thing about both the lasers and the sounds waves is
that they exert tiny forces. With the music during a pop concert, you can
really feel those forces in your stomach. With light, it's perhaps a bit harder
to imagine: after all, you don't experience a "thump" whenever you turn on the
light, nor do you feel the sunlight pushing you back on a sunny day. Yet even
light pushes against things a tiny bit, and this is actually used in the vacuum
of space to fly spacecraft, using "solar sails". At Lumicks, we harness those
tiny forces to push and pull on the cells and molecules that make up your body.
We can, for instance, "grab" a DNA molecule by its ends and stretch it, and
then watch how other molecules land on it to repair damage on the DNA.
Researchers learn invaluable things from such experiments about the processes
that lead to diseases, including cancer.

__CAF Team__: You are working with very small scales but very high resolutions.
We can imagine it takes quite a lot of data processing in real-time in order to
visualize the experiments and allow scientists to manipulate individual
proteins or cells while studying their behaviour. What are the biggest software
engineering challenges that you faced during initial product development?

__Lumicks Team__: Perhaps surprisingly, data volume is not our biggest
challenge. Our sensor data comes in at hundreds of MB per second, tops. The
real challenge lies in the large variety of hardware we need to talk to from
the software, and orchestrating all the hardware in a coordinated way, often
with strict timing requirements. We also have both a GUI and Python interface,
which can run simultaneously, and so need some careful synchronisation. Plus
each researcher using our software has their own specific needs for the
experiments they are doing, so we need a lot of flexibility in putting the
various modules together.

__CAF Team__: A big part of software engineering is constant learning about the
problem domain and iterating on a prototype until a robust solution emerges.
However, we can't iterate endlessly. Time to market is very crucial, even
critical in a startup like Lumicks. What brings the actor model, and CAF in
particular, to the table that helps you iterating faster and reaching a
sophisticated solution in reasonable time?

__Lumicks Team__: Our application does a lot of parallel processing and
multi-threading: there is all sorts of hardware I/O running in the background,
plus data processing, storage management, etc. CAF allows us to stay sane
amidst all the threads: the actor framework enforces a clear model of data
ownership, and saves us from a lot of mutex-related headaches. Not having to
worry about a whole class of subtle, hard-to-reproduce bugs is a life saver. On
top of that, the availability of compile-time checking of messaging interfaces
is very useful, and something where CAF has a tangible advantage over actor
implementations we've seen in other frameworks and languages.

__CAF Team__: How did you learn about CAF and what convinced you to use it over
other frameworks?

__Lumicks Team__: As a team we have used actors in other languages, such as
LabVIEW and Erlang, and found it to be a good model for parallel processing.
During the development of our control software it became clear that actors
would be a great fit and and so we undertook a review of C++ frameworks. CAF
came out as a clear winner and we have been using it ever since. The active
development of the library was a major factor for us, as well as its
flexibility and level of compile-time safety. __CAF Team__: How would you
summarize your experience with CAF so far? What was the first version you've
used? Were there any unexpected obstacles in adopting CAF or aha moments?

__Lumicks Team__: CAF has been working very well for us. We started on version
0.15.5, in the Fall of 2017. We had some small issues in the beginning, mostly
related to the lack of official shared library build in Windows, but otherwise
have been closely tracking the official releases. We haven't had any major
unexpected obstacles, so far, but finding the Qt mix-in was a
pleasant surprise, it made integrating with our UI components much easier! 

__CAF Team__: You develop all of your software in house with an international
team of software engineers. How do you introduce new team members to CAF and
how would you describe the learning curve?

__Lumicks Team__: We haven't found the learning curve for new team members to
be too challenging. As a team we have built a number of abstractions around CAF
and so this helps to reduce the surface area that new team members need to
learn. We also work with CAF on a single node, and use typed interfaces as much
as possible to make intentions clear and explicit . Processes within the team
also help, all of our code is thoroughly reviewed before merging and we do lots
of internal knowledge sharing, from working together on tricky problems to
fortnightly show and tell sessions on new and interesting developments.

__CAF Team__: What could CAF do better to smooth out the learning curve and
help developers being more productive?

__Lumicks Team__: Although the CAF documentation has improved since we first
started using the library, it is still an area where more work would help to
smooth out the learning curve, along with some more complex examples of CAF
usage. Improved tooling for debugging would also make a big impact on
productivity. 

__CAF Team__: Final question: if you were to decide what the next feature of
CAF would be, what would you have in mind?

__Lumicks Team__: One feature that would really help us as a team is better
tooling to understand message flows, for example to visual the flow of messages
between actors. It would help not only to debug issues during development but
also to optimise performance. We are also particularly interested in the
possibilities of the new streaming API, so seeing that move out of its
experimental state would also be great!

__CAF Team__: Thank you very much for this interview! We wish you and your team
all the great success in helping scientists around the world to
understand---and hopefully cure!---diseases such as cancer and Alzheimer's!

__Lumicks Team__: Thank you! If any of your readers are interested in working
with CAF to help scientists unlock new types of experiments in a small dynamic
team based in Amsterdam then check out our [careers
page](https://lumicks.com/careers/)!
