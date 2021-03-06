.. _usecase_smpi:

Simulating MPI Applications
===========================

Discover SMPI
-------------

SimGrid can not only :ref:`simulate algorithms <usecase_simalgo>`, but
it can also be used to execute real MPI applications on top of
virtual, simulated platforms with the SMPI module. Even complex
C/C++/F77/F90 applications should run out of the box in this
environment. In fact, almost all proxy apps provided by the `ExaScale
Project <https://proxyapps.exascaleproject.org/>`_ only require minor
modifications to `run on top of SMPI
<https://github.com/simgrid/SMPI-proxy-apps/>`_.

This setting permits to debug your MPI applications in a perfectly
reproducible setup, with no Heisenbugs. Enjoy the full Clairevoyance
provided by the simulator while running what-if analysis on platforms
that are still to be built! Several `production-grade MPI applications
<https://framagit.org/simgrid/SMPI-proxy-apps#full-scale-applications>`_
use SimGrid for their integration and performance testing.

MPI 2.2 is already partially covered: over 160 primitives are
supported. Some parts of the standard are still missing: MPI-IO, MPI3
collectives, spawning ranks, and some others. If one of the functions
you use is still missing, please drop us an email. We may find the
time to implement it for you.

Multi-threading support is very limited in SMPI. Only funneled
applications are supported: at most one thread per rank can issue any
MPI calls. For better timing predictions, your application should even
be completely mono-threaded. Using OpenMP (or pthreads directly) may
greatly decrease SimGrid predictive power. That may still be OK if you
only plan to debug your application in a reproducible setup, without
any performance-related analysis.

How does it work?
.................

In SMPI, communications are simulated while computations are
emulated. This means that while computations occur as they would in
the real systems, communication calls are intercepted and achived by
the simulator.

To start using SMPI, you just need to compile your application with
``smpicc`` instead of ``mpicc``, or with ``smpiff`` instead of
``mpiff``, or with ``smpicxx`` instead of ``mpicxx``. Then, the only
difference between the classical ``mpirun`` and the new ``smpirun`` is
that it requires a new parameter ``-platform`` with a file describing
the virtual platform on which your application shall run.

Internally, all ranks of your application are executed as threads of a
single unix process. That's not a problem if your application has
global variables, because ``smpirun`` loads one application instance
per MPI rank as if it was another dynamic library. Then, MPI
communication calls are implemented using SimGrid: data is exchanged
through memory copy, while the simulator's performance models are used
to predict the time taken by each communications. Any computations
occuring between two MPI calls are benchmarked, and the corresponding
time is reported into the simulator.

.. image:: /tuto_smpi/img/big-picture.svg
   :align: center	   

Describing Your Platform
------------------------

As a SMPI user, you are supposed to provide a description of your
virtual platform, that is mostly a set of simulated hosts and network
links with some performance characteristics. SimGrid provides a plenty
of :ref:`documentation <platform>` and examples (in the
`examples/platforms <https://framagit.org/simgrid/simgrid/tree/master/examples/platforms>`_
source directory), and this section only shows a small set of introductory
examples.

Feel free to skip this section if you want to jump right away to usage
examples.

Simple Example with 3 hosts
...........................

At the most basic level, you can describe your simulated platform as a
graph of hosts and network links. For instance:

.. image:: /tuto_smpi/3hosts.png
   :align: center

.. literalinclude:: /tuto_smpi/3hosts.xml
   :language: xml

Note the way in which hosts, links, and routes are defined in
this XML. All hosts are defined with a speed (in Gflops), and links
with a latency (in us) and bandwidth (in MBytes per second). Other
units are possible and written as expected. Routes specify the list of
links encountered from one route to another. Routes are symmetrical by
default.

Cluster with a Crossbar
.......................

A very common parallel computing platform is a homogeneous cluster in
which hosts are interconnected via a crossbar switch with as many
ports as hosts, so that any disjoint pairs of hosts can communicate
concurrently at full speed. For instance:

.. literalinclude:: ../../examples/platforms/cluster_crossbar.xml
   :language: xml
   :lines: 1-3,18-

One specifies a name prefix and suffix for each host, and then give an
integer range. In the example the cluster contains 65535 hosts (!),
named ``node-0.simgrid.org`` to ``node-65534.simgrid.org``. All hosts
have the same power (1 Gflop/sec) and are connected to the switch via
links with same bandwidth (125 MBytes/sec) and latency (50
microseconds).

.. todo::

   Add the picture.

Cluster with a Shared Backbone
..............................

Another popular model for a parallel platform is that of a set of
homogeneous hosts connected to a shared communication medium, a
backbone, with some finite bandwidth capacity and on which
communicating host pairs can experience contention. For instance:


.. literalinclude:: ../../examples/platforms/cluster_backbone.xml
   :language: xml
   :lines: 1-3,18-

The only differences with the crossbar cluster above are the ``bb_bw``
and ``bb_lat`` attributes that specify the backbone characteristics
(here, a 500 microseconds latency and a 2.25 GByte/sec
bandwidth). This link is used for every communication within the
cluster. The route from ``node-0.simgrid.org`` to ``node-1.simgrid.org``
counts 3 links: the private link of ``node-0.simgrid.org``, the backbone
and the private link of ``node-1.simgrid.org``.
	   
.. todo::

   Add the picture.

Torus Cluster
.............

Many HPC facilities use torus clusters to reduce sharing and
performance loss on concurrent internal communications. Modeling this
in SimGrid is very easy. Simply add a ``topology="TORUS"`` attribute
to your cluster. Configure it with the ``topo_parameters="X,Y,Z"``
attribute, where ``X``, ``Y`` and ``Z`` are the dimension of your
torus.

.. image:: ../../examples/platforms/cluster_torus.svg
   :align: center

.. literalinclude:: ../../examples/platforms/cluster_torus.xml
   :language: xml

Note that in this example, we used ``loopback_bw`` and
``loopback_lat`` to specify the characteristics of the loopback link
of each node (i.e., the link allowing each node to communicate with
itself). We could have done so in previous example too. When no
loopback is given, the communication from a node to itself is handled
as if it were two distinct nodes: it goes twice through the private
link and through the backbone (if any).

Fat-Tree Cluster
................

This topology was introduced to reduce the amount of links in the
cluster (and thus reduce its price) while maintaining a high bisection
bandwidth and a relatively low diameter. To model this in SimGrid,
pass a ``topology="FAT_TREE"`` attribute to your cluster. The
``topo_parameters=#levels;#downlinks;#uplinks;link count`` follows the
semantic introduced in the `Figure 1B of this article
<http://webee.eedev.technion.ac.il/wp-content/uploads/2014/08/publication_574.pdf>`_.

Here is the meaning of this example: ``2 ; 4,4 ; 1,2 ; 1,2``

- That's a two-level cluster (thus the initial ``2``).
- Routers are connected to 4 elements below them, regardless of its
  level. Thus the ``4,4`` component that is used as
  ``#downlinks``. This means that the hosts are grouped by 4 on a
  given router, and that there is 4 level-1 routers (in the middle of
  the figure).
- Hosts are connected to only 1 router above them, while these routers
  are connected to 2 routers above them (thus the ``1,2`` used as
  ``#uplink``).
- Hosts have only one link to their router while every path between a
  level-1 routers and level-2 routers use 2 parallel links. Thus the
  ``1,2`` that is used as ``link count``.

.. image:: ../../examples/platforms/cluster_fat_tree.svg
   :align: center

.. literalinclude:: ../../examples/platforms/cluster_fat_tree.xml
   :language: xml
   :lines: 1-3,10-


Dragonfly Cluster
.................

This topology was introduced to further reduce the amount of links
while maintaining a high bandwidth for local communications. To model
this in SimGrid, pass a ``topology="DRAGONFLY"`` attribute to your
cluster.

.. literalinclude:: ../../examples/platforms/cluster_dragonfly.xml
   :language: xml

.. todo::

   Add the image, and the documuentation of the topo_parameters.

Final Word
..........

We only glanced over the abilities offered by SimGrid to describe the
platform topology. Other networking zones model non-HPC platforms
(such as wide area networks, ISP network comprising set-top boxes, or
even your own routing schema). You can interconnect several networking
zones in your platform to form a tree of zones, that is both a time-
and memory-efficient representation of distributed platforms. Please
head to the dedicated :ref:`documentation <platform>` for more
information.

Hands-on!
---------

It is time to start using SMPI yourself. For that, you first need to
install it somehow, and then you will need a MPI application to play with.

Using Docker
............

The easiest way to take the tutorial is to use the dedicated Docker
image. Once you `installed Docker itself
<https://docs.docker.com/install/>`_, simply do the following:

.. code-block:: shell

   docker pull simgrid/tuto-smpi
   docker run -it --rm --name simgrid --volume ~/smpi-tutorial:/source/tutorial simgrid/tuto-smpi bash

This will start a new container with all you need to take this
tutorial, and create a ``smpi-tutorial`` directory in your home on
your host machine that will be visible as ``/source/tutorial`` within the
container.  You can then edit the files you want with your favorite
editor in ``~/smpi-tutorial``, and compile them within the
container to enjoy the provided dependencies.

.. warning::

   Any change to the container out of ``/source/tutorial`` will be lost
   when you log out of the container, so don't edit the other files!

All needed dependencies are already installed in this container
(SimGrid, the C/C++/Fortran compilers, make, pajeng and R). Vite being
only optional in this tutorial, it is not installed to reduce the
image size. 

The container also include the example platform files from the
previous section as well as the source code of the NAS Parallel
Benchmarks. These files are available under
``/source/simgrid-template-smpi`` in the image. You should copy it to
your working directory when you first log in:

.. code-block:: shell

   cp -r /source/simgrid-template-smpi/* /source/tutorial
   cd /source/tutorial

Using your Computer Natively
............................

To take the tutorial on your machine, you first need to :ref:`install
SimGrid <install>`, the C/C++/Fortran compilers and also ``pajeng`` to
visualize the traces. You may want to install `Vite
<http://vite.gforge.inria.fr/>`_ to get a first glance at the
traces. The provided code template requires make to compile. On
Debian and Ubuntu for example, you can get them as follows:

.. code-block:: shell

   sudo apt install simgrid pajeng make gcc g++ gfortran vite

To take this tutorial, you will also need the platform files from the
previous section as well as the source code of the NAS Parallel
Benchmarks. Just  clone `this repository
<https://framagit.org/simgrid/simgrid-template-smpi>`_  to get them all:

.. code-block:: shell

   git clone git@framagit.org:simgrid/simgrid-template-smpi.git
   cd simgrid-template-smpi/

If you struggle with the compilation, then you should double check
your :ref:`SimGrid installation <install>`.  On need, please refer to
the :ref:`Troubleshooting your Project Setup
<install_yours_troubleshooting>` section.

Lab 0: Hello World
------------------

It is time to simulate your first MPI program. Use the simplistic
example `roundtrip.c
<https://framagit.org/simgrid/simgrid-template-smpi/raw/master/roundtrip.c?inline=false>`_
that comes with the template.

.. literalinclude:: /tuto_smpi/roundtrip.c
   :language: c

Compiling and Executing
.......................
	      
Compiling the program is straightforward (double check your
:ref:`SimGrid installation <install>` if you get an error message):


.. code-block:: shell
		
  $ smpicc -O3 roundtrip.c -o roundtrip


Once compiled, you can simulate the execution of this program on 16
nodes from the ``cluster_crossbar.xml`` platform as follows:

.. code-block:: shell

   $ smpirun -np 16 -platform cluster_crossbar.xml -hostfile cluster_hostfile.txt ./roundtrip

- The ``-np 16`` option, just like in regular MPI, specifies the
  number of MPI processes to use. 
- The ``-hostfile cluster_hostfile.txt`` option, just like in regular
  MPI, specifies the host file. If you omit this option, ``smpirun``
  will deploy the application on the first machines of your platform.
- The ``-platform cluster_crossbar.xml`` option, **which doesn't exist
  in regular MPI**, specifies the platform configuration to be
  simulated. 
- At the end of the line, one finds the executable name and
  command-line arguments (if any -- roundtrip does not expect any arguments).



We will use following simple MPI program, roundtrip.c, in which the processes pass around a message and print the elpased time:

 
..  LocalWords:  SimGrid
