.. First introduction

What is SimGrid
===============

SimGrid is a framework to simulate distributed computer systems.

It can be used to either assess abstract algorithms, or to profile and
debug real distributed applications.  SimGrid enables studies in the
domains of (data-)Grids, IaaS Clouds, Clusters, High Performance
Computing, Volunteer Computing and Peer-to-Peer systems.

Technically speaking, SimGrid is a library. It is neither a graphical
interface nor a command-line simulator running user scripts. The
interaction with SimGrid is done by writing programs with the exposed
functions to build your own simulator.

SimGrid offers many features, many options and many possibilities. The
documentation aims at smoothing the learning curve. But nothing's
perfect, and this documentation is really no exception here. Please
help us improving it by reporting any issue that you see and
proposing the content that is still missing.

SimGrid is a Free Software distributed under the LGPL licence. You are
thus welcome to use it as you wish, or even to modify and distribute
your version (as long as your version is as free as ours). It also
means that SimGrid is developed by a vivid community of users and
developers. We hope that you will come and join us!

SimGrid is the result of almost 20 years of research from several
groups, both in France and in the USA. It benefited of many funding
from various research instances, including the ANR, Inria, CNRS,
University of Lorraine, University of Hawai'i at Manoa, ENS Rennes and
many others. Many thanks to our generous sponsors!

Typical Study based on SimGrid
------------------------------

Any SimGrid study entails the following components:

 - The studied **Application**. This can be either a distributed
   algorithm described in our simple APIs, or a full featured real
   parallel application using for example the MPI interface
   @ref application "(more info)".

 - The **Virtual Platform**. This is a description of a given
   distributed system (machines, links, disks, clusters, etc). Most of
   the platform files are written in XML althrough a Lua interface is
   under development.  SimGrid makes it easy to augment the Virtual
   Platform with a Dynamic Scenario where for example the links are
   slowed down (because of external usage), the machines fail. You
   have even support to specify the applicative workload that you want
   to feed to your application  @ref platform "(more info)".

 - The application's **Deployment Description**. In SimGrid
   terminology, the application is an inert set of source files and
   binaries. To make it run, you have to describe how your application
   should be deployed on the virtual platform. You need to specify
   which process is mapped on which host, along with their parameters
   @ref deployment "(more info)".

 - The **Platform Models**. They describe how the virtual platform
   reacts to the actions of the application. For example, they compute
   the time taken by a given communication on the virtual platform.
   These models are already included in SimGrid, and you only need to
   pick one and maybe tweak its configuration to get your results
   @ref models "(more info)".

These components are put together to run a **simulation**, that is an
experiment or a probe. The result of one or many simulation provides
an **outcome** (logs, visualization, statistical analysis) that help
answering the **question** targeted by this study.

The questions that SimGrid can solve include the following:

 - **Compare an Application to another**. This is the classical use
   case for scientists, who use SimGrid to test how the solution that
   they contribute compares to the existing solutions from the
   literature.

 - **Design the best Virtual Platform for a given Application.**
   Tweaking the platform file is much easier than building a new real
   platform for testing purpose. SimGrid also allows co-design of the
   platform and the application by modifying both of them.

 - **Debug Real Applications**. With real systems, is sometimes
   difficult to reproduce the exact run leading to the bug that you
   are tracking. SimGrid gives you experimental reproducibility,
   clairevoyance (you can explore every part of the system, and your
   probe will not change the simulated state). It also makes it easy
   to mock some parts of the real system that are not under study.

Depending on the context, you may see some parts of this process as
less important, but you should pay close attention if you want to be
confident in the results coming out of your simulations. In
particular, you should not trust blindly your results but always
strive to double-check them. Likewise, you should question the realism
of your input configuration, and we even encourage you to doubt (and
check) the provided performance models.

To ease such questionning, you really should logically separate these
parts in your experimental setup. It is seen as a very bad practice to
merge the application, the platform and the deployment all together.
SimGrid is versatile and your milleage may vary, but you should start
with your Application specified as a C++ or Java program, using one of
the provided XML platform file, and with your deployment in a separate
XML file.

SimGrid Execution Modes
-----------------------

Depending on the intended study, SimGrid can be run in several execution modes.

** **Simulation Mode**. This is the most common execution mode, where you want
to study how your application behaves on the virtual platform under
the experimental scenario.

In this mode, SimGrid can provide information about the time taken by
your application, the amount of energy dissipated by the platform to
run your application and the detailed usage of each resource.

** **Model-Checking Mode**. This can be seen as a sort of exhaustive
testing mode, where every possible outcome of your application is
explored. In some sense, this mode tests your application for all
possible platforms that you could imagine (and more).

You just provide the application and its deployment (amount of
processes and parameters), and the model-checker will litterally
explore all possible outcomes by testing all possible message
interleavings: if at some point a given process can either receive the
message A first or the message B depending on the platform
characteristics, the model-checker will explore the scenario where A
arrives first, and then rewind to the same point to explore the
scenario where B arrives first.

This is a very powerful mode, where you can evaluate the correction of
your application. It can verify either **safety properties** (asserts)
or **liveless properties** stating for example that if a given event
occures, then another given event will occur in a finite amount of
steps. This mode is not only usable with the abstract algorithms
developed on top of the SimGrid APIs, but also with real MPI
applications (to some extend).

The main limit of Model Checking lays in the huge amount of scenarios
to explore. SimGrid tries to explore only non-redundent scenarios
thanks to classical reduction techniques (such as DPOR and stateful
exploration) but the exploration may well never finish if you don't
carefully adapt your application to this mode.

A classical trap is that the Model Checker can only verify whether
your application fits the provided properties, which is useless if you
have a bug in your property. Remember also that one way for your
application to never violate a given assert is to not start at all
because of a stupid bug.

Another limit of this mode is that it does not use the performance
models of the simulation mode. Time becomes discrete: You can say for
example that the application took 42 steps to run, but there is no way
to know the amount of seconds that it took or the amount of watts that
it dissipated.

Finally, the model checker only explores the interleavings of
computations and communications. Other factors such as thread
execution interleaving are not considered by the SimGrid model
checker.

The model checker may well miss existing issues, as it computes the
possible outcomes *from a given initial situation*. There is no way to
prove the correction of your application in all generality with this
tool.

** **Benchmark Recording Mode**. During debug sessions, continuous
integration testing and other similar use cases, you are often only
interested in the control flow. If your application apply filters to
huge images split in small blocks, the filtered image is probably not
what you are interested in. You are probably looking for a way to run
each computation kernel only once, save on disk the time it takes and
some other metadata. This code block can then be skipped in simulation
and replaced by a synthetic block using the cached information. The
virtual platform will take this block into account without requesting
the real hosting machine to benchmark it.

SimGrid Limits
--------------

This framework is by no means the perfect holly grail able to solve
every problem on earth.

** **SimGrid scope is limited to distributed systems.** Real-time
multithreaded systems are not in the scope. You could probably tweak
SimGrid for such studies (or the framework could possibily be extended
in this direction), but another framework specifically targeting this
usecase would probably be more suited.

** **There is currently no support for IoT studies and wireless networks**.
The framework could certainly be improved in this direction, but this
is still to be done.

** **There is no perfect model, only models adapted to your study.**
The SimGrid models target fast, large studies yet requesting a
realistic results. In particular, our models abstract away parameters
and phenomenon that are often irrelevant to the realism in our
context.

SimGrid is simply not intended to any study that would mandate the
abstracted phenomenon. Here are some **studies that you should not do
with SimGrid**:

 - Studying the effect of L3 vs L2 cache effects on your application
 - Comparing variantes of TCP
 - Exploring pathological cases where TCP breaks down, resulting in
   abnormal executions.
 - Studying security aspects of your application, in presence of
   malicious agents.

SimGrid Success Stories
-----------------------

SimGrid was cited in over 1,500 scientific papers (according to Google
Scholar). Among them
`over 200 publications <http://simgrid.gforge.inria.fr/Usages.php>`_
(written by about 300 individuals) use SimGrid as a scientific
instrument to conduct their experimental evaluation. These
numbers do not count the articles contributing to SimGrid.
This instrument was used in many research communities, such as
`High-Performance Computing <https://hal.inria.fr/inria-00580599/>`_,
`Cloud Computing <http://dx.doi.org/10.1109/CLOUD.2015.125>`_,
`Workflow Scheduling <http://dl.acm.org/citation.cfm?id=2310096.2310195>`_,
`Big Data <https://hal.inria.fr/hal-01199200/>`_ and
`MapReduce <http://dx.doi.org/10.1109/WSCAD-SSC.2012.18>`_,
`Data Grid <http://ieeexplore.ieee.org/document/7515695/>`_,
`Volunteer Computing <http://www.sciencedirect.com/science/article/pii/S1569190X17301028>`_,
`Peer-to-Peer Computing <https://hal.archives-ouvertes.fr/hal-01152469/>`_,
`Network Architecture <http://dx.doi.org/10.1109/TPDS.2016.2613043>`_,
`Fog Computing <http://ieeexplore.ieee.org/document/7946412/>`_, or
`Batch Scheduling <https://hal.archives-ouvertes.fr/hal-01333471>`_
`(more info) <http://simgrid.gforge.inria.fr/Usages.php>`_.

If your platform description is accurate enough (see
`here <http://hal.inria.fr/hal-00907887>`_ or
`there <https://hal.inria.fr/hal-01523608>`_),
SimGrid can provide high-quality performance predictions. For example,
we determined the speedup achieved by the Tibidabo Arm-based
cluster before its construction
(`paper <http://hal.inria.fr/hal-00919507>`_). In this case,
some differences between the prediction and the real timings were due to
misconfiguration or other problems with the real platforms. To some extent,
SimGrid could even be used to debug the real platform :)

SimGrid is also used to debug, improve and tune several large
applications.
`BigDFT <http://bigdft.org>`_ (a massively parallel code
computing the electronic structure of chemical elements developped by
the CEA), `StarPU <http://starpu.gforge.inria.fr/>`_ (a
Unified Runtime System for Heterogeneous Multicore Architectures
developped by Inria Bordeaux) and
`TomP2P <https://tomp2p.net/dev/simgrid/>`_ (a high performance
key-value pair storage library developped at University of Zurich).
Some of these applications enjoy large user communities themselves.

Where to proceed next?
----------------------

Now that you know about the basic concepts of SimGrid, you can give it
a try. If it's not done yet, first :ref:`install it <install>`. Then,
proceed to the section on @ref application "describing the application" that
you want to study.
