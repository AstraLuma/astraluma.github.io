---
title: Distributed Unix
---

Goal
----
Provide POSIX-compatible system distributable across thousands of machines.

Or: What if all the servers of Google were a single usable system?

Unsolved Problems
=================
* Global numerical ID (UID, GID, INODE, PID) exhaustion
  * `uid_t`, `gid_t`, and `pid_t` are distinct types, so may be very large integers
  * Files are tracked by `dev_t` and `ino_t` and so might also be very large numbers
    * Utilities use the device to namespace inodes for hardlink tracking?
    * Does POSIX allow hardlinks to span devices?
    * Can the device of a file refer to a replication group in the global SAN, or must it refer to which SAN has the file?
      * This could allow unreplicated filesystem sources (eg, backed by individual nodes) for eg /tmp
* Fork latency
* Packet (eg, UDP) daemons
* Database daemons
* Filesystem daemons (a la plan 9)
* Distributed file system
* Network failure

The big problem is: How do you communicate changes in system state (eg process status) to interested nodes without swamping the entire network with status messages.

Robustness and Availability
===========================
The overall system should be robust against node failures. IE, if a single node has an error, power failure, or catastrophic disaster, the overall system should be able to recover. The particular resources (hardware, processes) backed by that node may become unavailable, but it should not cascade to a system crash.

The file system should replicated so that _n_ node failures do not prevent resource access. In particular, virtual inodeds (devices, sockets, FIFOs) should never be lost, given their tendency to be critical to system operation. VFS mounts are backed by daemons, so their availability is tied to the robustness of the daemon.

Compute Fabric
==============
The unit of computation is the process. Fabric does not recognize threads.

Hardware
--------
The compute fabric is made of a series of nodes sharing a LAN.

Each node is an independent PC (x86 or ARM based). The entire fabric is a single architecture.

(Should differing processor features be allowed? Is it feasible to move a process from a processor with less features to a processor with more?)

(Not sure if dunix should be a stand-alone operating system or a layer on top of a non-distributed kernel. I'm writing as if its a bare-metal OS, but it's feasible to adapt it to be a wine-like layer.)

### CPU Features

On start, a process is tagged with which CPU features it uses (eg, ARM instruction sets, x86 extensions). A process can only move to a node with a superset of its CPU features.

While strictly speaking, this could allow ARM and x86 nodes to coexist on the same system, this isn't terribly practical.

Node Management
---------------
* Node management is Peer-to-Peer
* Each node has a list of peer nodes and maintains the status of each of its peers
* The status information is things primarily involved in fork decisions (eg CPU load, distance to FS pieces)

To Move Between Nodes
---------------------
* Have: PID, Node to move to
1. Send STOP signal to process, making it unschedulable.
2. Package memory pages, CPU context, and kernel structures
3. Transmit to new node
4. New node unpacks process
5. Send CONT signal to process

Processes may move arbitrarily between nodes, depending on load, file system concerns, etc.

Unmovable Processes
-------------------
* Direct FD to local hardware
* Shared memory with any other process

(mmap() to files is pretty difficult with distributed FS.)

On fork
-------
1. Fork locally
2. If local machine can handle additional load, break
3. Immediately send STOP to child before it is scheduled
4. From list of peers, select node to send child to
5. Move child to new node, following Move Between Nodes algorithm

Streams
=======

Because of process shuffling, part of the fabric infrastructure is the ability to move streams (File Descriptors) between machines. Various processes have FDs spanning between them, which may be local to a node or between nodes.

Networking
==========
* Edge routers are responsible for presenting outside open ports
* Internal network is IPv6 (autoconf)

Edge routers are psuedo-nodes: They have a list of peers but do not participate in the fabric

Stream Daemons
--------------
* On socket listen, process becomes fork template.
* Listen never returns on parent process
* On connection, fork is created
* Parent process accepts signals

(Alternative: inetd)

This was chosen on the assumption that fork overhead is much lower than reinitialization, configuration reload, etc. A daemon has the flexibility to do initialization and management (through signals) from a central process but still take advantage of load balancing compute fabric.

### On Listen
1. Registration is sent to edge routers, which are responsible for load balancing

### On Connection (TCP)
1. Edge router accepts the connection and creates an FD
2. Edge router gives the node with the parent template the connection FD
3. Parent node forks the template process
4. The child process resumes from the listen call, receiving the connection FD

### On Connection (Unix, FIFO)
The inode stores a reference to the listening (parent/template) PID.

1. Client process creates an FD
2. Client node sends the FD to the parent node
3. Parent node forks the template process
4. Child process resumes from listen call, receiving the connection FD

Filesystem
==========
This is largely undetermined.

* Storage fabric and compute fabric same hardware?
* Storage fabric is basically SAN
* Contains real files, directories, and data required to make connections (Unix, FIFO)
* If following plan 9, daemons may implement VFS.
  * This is actually very problematic based on forking model of daemons
* Full ACLs (MAC) and real locks are probably a requirement
* Is eventual consistency allowable in the file system?

Workstations
============
Graphical workstations can work in this context.

* A process holding certain kinds of FDs (eg, Wayland with open DRI hardware) 
  makes it unmovable (ie it is tied to the node)
* A process connected to an unmovable process is weighted against moving away from the node (eg, graphical apps)
* Userspace very similar to old timeshare systems.
* VT works similarly: init ties terminal FD to login process

I haven't figured out how display/login managers get local hardware FDs.

* `/dev/node/*`? Probably has some pretty big security concerns.

Databases
=========
* Nodes and implicit forking completely interfere with parallelism.
* No idea how to rectify.
* Real filesystem locks may help? (Possible as long as process death releases lock)
* Ideally, databases map to filesystem (eg, plan 9), but querying is not a thing in this model?


Numeric Identifiers
===================

uid_t and gid_t
---------------
32-bit numbers are likely sufficient. I believe that 4 billion groups or users are unlikely to be found on a single system.

dev_t
-----
This is used to identify both node devices (keyboards, displays, etc) and file sources (disk drives, storage clusters). Therefore, it will likely need to be a structured number including if it's a real or virtual device, the node providing it (real only), and some other identifiers.

The idea of major and minor device numbers are all but lost in this system.

pid_t
-----
A process is only on one node at a time. It may be useful assign an 'authoritive' node (probably the origin) to the process and encode it in the PID. However, this requires the node to always service the process, even if it has given it to another node. It may also act as a bottle neck and in the event of failure, cause the process to be orphaned and lost.

However, we do not want every node tracking every process, either.

We must also prevent two processes from ever getting the same ID.

Further Reading
===============
* Inferno: Distributed Plan 9-based operating system able to be used across a mixed-architecture fabric
* [Approaches to Distributed UNIX Systems](http://hdl.handle.net/10022/AC:P:11720): Paper from 1986 discussing approaches to distributed Unix
