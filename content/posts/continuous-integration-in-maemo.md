---
title: "Continuous Integration in Maemo"
date: 2019-10-28T21:00:45+02:00
draft: false
noSummary: false
---

In the previous post about Maemo I made a quick review of its
human organization. Now I'd like to add a couple of words about
its release process.

<!--more-->

### BIFH

Nowadays embedded development mostly associates with Yocto and usually
release processes are defined by the peculiarities of Yocto. In Maemo
it was all different, because e.g. the metadata of source packages was kept
not in receipts, but in a SQL-database, an integration process into a product
was initiated not by pull requests to a Yocto-layer, but by posting a message
formatted in a special DSL into Maemo's release system called BIFH.
BIFH is a proprietary system that has never been opened by Nokia, though
there was such [intent](http://bifh.org/wiki/). It resembles
Suse's [Open Build Service](https://openbuildservice.org/) a bit,
but OBS knows nothing about integration requests.

I'd say that BIFH was a reflection of Maemo's organization and grew
organically from it and was never finished. I'll try to present somewhat
idealistic view on it, because some concepts have never been implemented
in BIFH and were buried in backlog.

First of all let's introduce BIFH's speak.

### Terminology

__Software Platform__ is a set of software components belonging to various
architectural layers used by product vendors (i.e. Nokia) to build their
products upon.

__Software Platform Taxonomy__ is a scheme of hierarchical classification of
pieces of code. The scheme tends to evolve from platform to platform. It may
be defined in the same way as an organization is structured: projects &rarr;
component teams &rarr; source packages. Or it may reflect a software
architecture more or less. The common rule is that binary packages are leaves
in the structure always. Then between a binary and the root there might be
several levels of intermediate nodes of different types belonging to one or
more hierarchies: i.e. architectural hierarchy, copyright hierarchy,
deployment configuration hierarchy, certification hierarchy, hierarchy in
terms of a bug tracking system.

Different actors are interested in different aspects of the taxonomy. Legal
managers may be interested to see source packages with no copyright owner
set. Architects would want to overview platform’s architecture or to move
a source package from one subsystem to another.

Some actions on nodes may trigger a process of approval of the action according
to a specified workflow.

Also Access Control Lists can be applied to the taxonomy.

__Deployment Units__ of various configurations are final results of a software
vendor which are meant to be ready for production or sale or distribution.
Examples: flashable images, distribution packages, ISO images, VM images.
The possibility to change the configuration of deployment units easily
(to add or to remove a certain feature, closed source code, packages with debug
symbols or whatever else) is the cornerstone for easy productization.

__Build variant__ is a combination of a toolchain and compiler settings
(e.g. code instrumentalization on/off).

__Package repository__ is a storage of source and binary packages used to
produce a deployment unit or to bootstrap a build environment. A repository
may contain packages built for different architectures and build variants.
Packages must be built against other packages that already reside in the
repository. A package repository may be structured in a way that for some of
its parts ACL can be applied (this is the enabler for collaboration with
external subcontractors signed NDA not covering all the source code, e.g.
at that time it were sources of Macromedia Flash, Opera, Skype etc).
I suspect Yocto and OBS lack this feature even now, but at Nokia
ACLs were a must.

__Package Repository Management System__ is a system responsible for creation,
updating and querying package repositories (e.g. serving queries to produce
changelog diffs between two arbitrary updates of a repository).
As an open source analogue of such system [aptly](https://www.aptly.info/)
looks promising.

__Baseline__ is a snapshot of a package repository.

__Build Environment__ is a combination of a build variant and a baseline
(optionally with some constraints applied, i.e. copyright constraints)
used to transform sources into binaries.

__Product Branch__ is a fork-able sequence of baselines. A product branch might
be quantized at the scale of weekly releases or on per integration request
basis.

__Release__ is a baseline marked as a “release” and a set of deployment
units built against the baseline.

__VCS Runner__ is a daemon which monitors source code repositories associated
with registered in a platform taxonomy source packages and emits events upon
new commits or tags which may trigger build processes or even integration
processes of new versions of the source code. BIFH used polling to
detect new updates in version control systems, because it was running
behind a corporate proxy.

__Validation__ is a collection of various activities such as API checks,
ABI compatibility checks, source code checks (static code analysis for example),
build log checks, packaging checks, installation log checks, copyright checks,
deployment unit checks including automated functional and unit testing,
**Seamless Software Updates** and so on and so forth...

### A day in BIFH's life

So, the platform taxonomy was served by a Django-based web application
(a package registry).
Any package that could become a part of an end product was
registered in the app's database. The registered metadata included
license, copyright owner, required certifications, VCS URL, ACLs etc.

But product configurations were defined through dependencies of
meta-packages and any change in a configuration had to follow the
same workflow that was used for other packages.

Let's imagine we want to include an ICQ-plugin to Maemo's
communication framework Telepathy. Then we need to register the plugin's
metadata in the package registry. As soon as we add a VCS URL(s) and mark
it enabled the VCS Runner starts to monitor the source repository
for new tags or commits. Upon a new tag the VCS Runner generates a
request (from a template also added to the package registry) to start an
integration workflow for the new package release. The steps (all optional) are

1. download sources;
2. run a configurable set of source checks and validations;
3. build binary packages in a build environment either defined by
   the target product (in case of a product-ready release) or
   tweaked by additional options attached to the registered VCS URL
   (in case of a build for development or testing purposes);
4. run a configurable set of binary checks and validations;
5. publish binaries in the request's repository;
6. publish binaries in a configurable set of package repositories;
7. build a configurable set of deployment units with the newly
   built binaries included;
8. run a configurable set of tests and validations;
9. optionally indicate that the packages are ready for product integration
   tests.

The format of integration requests evolved from a simple enumeration of
package names to a rather complex DSL to satisfy all possible corner
cases. Naturally requests could be submitted manually either
through a web interface or by mail. Sometimes a manual submission
was the only way to make an integration request successful i.e.
in case of two source packages (`A` and `B`) developed in two different
Git repositories when a new version of `A` depended on a new version of `B`
whereas the new version of `B` was incompatible with the current
version of `A`. This way we can guaranty that the product repository
are always in good state and not broken.

Release candidates were built by sending manual requests too, but with
steps for building and checking packages excluded.

Also this approach helped a lot when testers reported bugs against
a test build. Since every build (including builds for product release
candidates) could be referenced by its request ID
developers could install debug symbols from the request's binary repository.
And crash-tools could even automatically produce stack traces for
collected crash dump files and mail them to the request's originator or
automatically create a bug in bugzilla. OBS lacks this feature and
often by the time a crash happens the corresponding debug symbols get
overridden by a new build.
