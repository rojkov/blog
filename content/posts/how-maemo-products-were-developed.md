---
title: "How Maemo Products Were Developed"
date: 2019-10-21T20:29:00+03:00
draft: false
noSummary: false
---

Any project that goes beyond being a mere personal hobby is always a
collective effort where people work in concert. And these people
working in concert is the foundation on which any product is built.
Means of production are useless without people, but people need to
be organized, their relations have to be structured to transform
the collective effort into a meaningful outcome.

I used to be a part of the [Maemo](https://en.wikipedia.org/wiki/Maemo)
project at Nokia. In this post I'd like to write down what I still remember
about the ways the products like [Nokia N9](https://en.wikipedia.org/wiki/Nokia_N9)
were developed.

<!--more-->

Usually people's relations are structured with agreed policies.
The policies are enforced either through organizational hierarchies or
special tools. The more policy enforcement relies on tools the less
is the need for managers managing the hierarchies.

At Nokia the hierarchical structure was huge and had more than one
dimension.

Perhaps it makes sense to start with *platforms*. It is a very loaded
term, but at Nokia "platform" was used mostly to denote a base operating
system on which products were built. There were [S40](https://en.wikipedia.org/wiki/Series_40),
[S60](https://en.wikipedia.org/wiki/Series_60), WindowsPhone and
Maemo platforms. So strictly speaking Maemo was not a project,
but platform. Every platform had its own big boss.

Then we had *programs* for the platforms. A program was led by a
program manager. Programs were defined by base hardware configurations.

And in the scope of a program (meaning on top of a base hardware configuration)
*products* where developed, that is devices of slightly different hardware
versions and flashable images for them. A product also included "maintenance"
that is software updates for the released device. Obviously every product
had its own product manager reporting to the program manager and responsible
for her product's timely releases with certain functionality.

In addition to that there were configuration managers responsible for
product configurations targeting different markets. For example the Maps
application for Nokia N9 selling in Iran knew nothing about Israel,
users in China saw Taiwan as PRC's territory and so on. In some funny
markets the Camera app must produce "shutter" sound when taking a
picture, so the setting for disabling the sound must be excluded.

Non-managers (also known as developers) were organized in *teams*.
Every team was responsible for development of one subsystem of
the platform. Examples are the Real-time Communication team, the Applications
team (Calculator, Gallery and others), the Application Framework team
(UI and non-UI libraries), the Imaging team (Camera), the Browser team,
the Multimedia team (sound processing and playback), the System Software
team (Kernel and hardware adaptation) and others. Besides that there were
auxiliary teams that didn't deliver any functionality to products, but
supported the main teams. Without them products would be never released:

* the SDK team (toolchains, cross-compilers);
* the Performance team (optimizations and fixing performance issues);
* the Release & Integration team. Probably this was the most important
  team which developed the policy enforcement tools mentioned in
  the beginning of the post, maintained Maemo's CI system when Martin
  Fowler had not yet coined the term and started to practice GitOps
  long before WeaveWorks. All this made it possible to integrate the
  deliverables of all other teams automatically (to some extent)
  into an end product. Later the team members formed the core of Jolla.

Naturally every team was led by a line manager who did budgeting, facilitating
and performance assessments.

Inside the teams there were project managers who served as an interface
between team members and product managers (teams delivered features to
multiple products at the same time) and communicated with other teams
because e.g. Applications depend on Application Framework, Phone depends
on Multimedia and Camera (for video calls), almost everything depends
Hardware Adaptation and everybody needs SDK.

Also I'd like to mention product QA, UX designers, industrial designers,
sourcing, sales, marketing.

In other words in 2000s it could cost a fortune to develop new
mobile phones. But the good news is that a substantial part of the
developed code was made open. Combined with the experience
accumulated by the people it enabled Jolla to develop its own
mobile platform way cheaper. Needless to say Jolla's organizational
structure was almost flat, it relied heavily on its automated
release process.
