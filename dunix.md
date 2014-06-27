---
title: Distributed Unix
---

Goal
----
Provide POSIX-compatible system distributable across thousands of machines.

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

Compute Fabric
==============
The unit of computation is the process. Fabric does not recognize threads.

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

Workstations
============
Graphical workstations can work in this context.

* A process holding certain kinds of FDs (eg, Wayland with open DRI hardware) 
  makes it unmovable (ie it is tied to the node)
* A process connected to an unmovable process is weighted against moving away from the node (eg, graphical apps)
* Userspace very similar to old timeshare systems.
* VT works similarly: init ties terminal FD to login process

I haven't figured out how display/login managers get local hardware FDs.

* `/dev/node/*` ? Probably has some pretty big security concerns.

Databases
=========
* Nodes and implicit forking completely interfere with parallelism.
* No idea how to rectify.
* Real filesystem locks may help? (Possible as long as process death releases lock)
* Ideally, databases map to filesystem (eg, plan 9), but querying is not a thing in this model?
