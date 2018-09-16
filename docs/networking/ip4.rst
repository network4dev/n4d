====
IPv4
====

NAT44
-----
Network Address Translation is a technique used to change, or translate,
the source or destination IPv4 address. This is not an exclusive "or" as both
address could be translated. This document is iterative in that each use case
is described and demonstrated sequentially and in small batches.

Definitions
^^^^^^^^^^^
Master these terms before continuing:

* inside: The network *behind* the NAT point, often where source
  addresses are translated from private to public.
* outside: The network *in front of* the NAT point. Often packets in
  this network hae publicly routable source IP addresses.
* state: The information stored in the NAT device about each translation.
  NAT state is typically stored in a table containing pre and post NAT
  addressing, and sometimes layer-4 port/protocol information as well.
* pool: A range of addresses which can be used as outputs from the NAT
  process. Addresses in a pool are dynamic resources which are allocated
  on-demand when hosts need to traverse a NAT and require different IPs.

In Cisco-speak, some additional terms are defined. These seem confusing at
first, but are a blessing in disguise because they are unambiguous.

* Inside local: IP on the *inside* network from the perspective of other
  hosts on the *inside* network. This is often a private RFC 1918 IP
  address and is seen by other hosts on the inside as such.
* Inside global: IP on the *inside* network from the perspective of
  hosts on the *outside* network. This is often a publicly routable
  address which represents the inside host to outside hosts.
* Outside global: IP on the *outside* network from the perspective of other
  hosts on the *outside* network. This is often a publicly routable
  address and is seen by other hosts on the outside as such.
* Outside local: IP on the *outside* network from the perspective of
  hosts on the *inside* network. This is usually the same as the outside
  global since the all hosts, whether inside or outside, view outside hosts
  as having a single, public IP address (but not always).

Notes
^^^^^
Things to keep in mind:

* All demonstrations assume Cisco Express Forwarding (CEF) has been disabled
  which allows IP packet debugging (routing lookups) to be shown. This is
  not supported on newer Cisco routers, so older devices running 12.4T are
  used in these demonstrations.

Static One-to-one (1:1) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
The simplest type of NAT is 1:1 NAT where a single source IP is mapped to
another. This technique is useful in the following cases:

* Inside host should not or cannot share a source IP with other hosts
* Custom IP protocols are used (not TCP, UDP, or ICMP)
* Outside-to-inside unsolicited traffic flow is required
* Address traceability is required, typically for security audits

When traffic traverses the NAT, only the source IP changes. In this example,
inside hosts ``192.0.2.1`` and ``192.0.2.2`` are mapped to ``203.0.113.1``
and ``203.0.113.1``, respectively. Cisco IOS syntax is below::

  # ip nat inside source static (inside_local) (inside_global)
  ip nat inside source static 192.0.2.1 203.0.113.1
  ip nat inside source static 192.0.2.2 203.0.113.2

Enabling NAT (``debug ip nat``) and IP packet (``debug ip packet``) debug
reveals how this technology works. Follow the order of operations:

1. Initial packet arrives from ``192.0.2.1`` to ``198.51.100.42`` (not in debug)
2. NAT device performs routing lookup towards ``198.51.100.42``
3. NAT device translates source from ``192.0.2.1`` to ``203.0.113.1``
4. NAT device forwards packet to ``198.51.100.1`` in the outside network
5. Return packet arrives from ``198.51.100.42`` to ``203.0.113.1`` (not in debug)
6. NAT device translates destination from ``203.0.113.1`` to ``192.0.2.1``
7. NAT device perform routing lookup towards ``192.0.2.1``
8. NAT device forwards packet to ``192.0.2.1`` in the inside network

Some information has been hand-editted from the debug for clarity::

  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: s=192.0.2.1->203.0.113.1, d=198.51.100.1
  IP: s=203.0.113.1, d=198.51.100.1, g=203.0.113.253, len 100, forward
  NAT*: s=198.51.100.1, d=203.0.113.1->192.0.2.1
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 100, forward

See if you can work through the following example on your own, explaining
each step, for the second host ``192.0.2.2``. Say each step aloud or hand-write
it on paper. Mastering the NAT order of operations is essential::

  IP: tableid=0, s=192.0.2.2, d=198.51.100.2, routed via RIB
  NAT: s=192.0.2.2->203.0.113.2, d=198.51.100.2
  IP: s=203.0.113.2, d=198.51.100.2, g=203.0.113.253, len 100, forward
  NAT*: s=198.51.100.2, d=203.0.113.2->192.0.2.2
  IP: tableid=0, s=198.51.100.2, d=192.0.2.2, routed via RIB
  IP: s=198.51.100.2, d=192.0.2.2, g=192.0.2.253, len 100, forward

The main drawbacks of this solution are scalability and manageability. A
network with 10,000 hosts would require 10,000 dedicated addresses with
no sharing, which limits scale. Managing all these NAT statements, though
made massively simpler with modern network automation, is still burdensome
as someone must still maintain the mapping source of truth.

Dynamic One-to-one (1:1) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Rather than identify every inside global address and its corresponding
mapping to an inside local address, one can use a NAT pool to define a
collection of addresses for dynamic allocation when traffic traverses the
NAT. It has the following benefits:
* Arbitrary pool size coupled with arbitrary inside host list
* Easy management and configuration
* Dynamically-allocated state

When traffic traverses the NAT, only the source IP changes. In this example,
inside hosts within ``192.0.2.0/25`` are dynamically mapped to an available
address in the pool ``203.0.113.0/27``. Note that the two network masks do
not need to match, and that the ACL can match any inside local address.
Cisco IOS syntax is below::

  # ACL defines the inside local addresses (pre NAT source)
  ip access-list standard ACL_INSIDE_SOURCES
   permit 192.0.2.0 0.0.0.127

  # Pool defines the inside global addresses (post NAT source)
  ip nat pool POOL_203 203.0.113.12 203.0.113.31 prefix-length 27

  # Binds the inside local list to the inside global pool for translation
  ip nat inside source list ACL_INSIDE_SOURCES pool POOL_203

The output from NAT and IP packet debugging shows an identical flow from the
previous section. The process has not changed, but the manner in which inside
local addresses are mapped to inside global addresses is fully dynamic::

  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: s=192.0.2.1->203.0.113.12, d=198.51.100.1
  IP: s=203.0.113.12, d=198.51.100.1, g=203.0.113.253, len 100, forward
  NAT: s=198.51.100.1, d=203.0.113.12->192.0.2.1
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 100, forward

Like the static 1:1 solution, this solution still requires a large number of
publicly routable (post NAT) addresses. Though the administrator does not
have to manage the addresses individually, simply obtaining public IPv4
addresses is a challenge given its impending exhaustion. The solution can
optionally allow outside-to-inside traffic but only after the NAT state
has already been created. The solution does not provide traceability
between inside local and inside global *unless* the administrator specifically
captures the NAT state table through shell commands, logging, or some
other means.
the

Static One-to-one (1:1) Inside Network NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This option is a hybrid of the previous static and dynamic 1:1 NAT tehcniques.
The solution requires one inside local prefix and one inside global prefix,
both of the same prefix length, to be mapped at once. The solution can be
repeated for multiple inside local/global prefix pairs.

It has the best of both worlds:
* Inside host should not or cannot share a source IP with other hosts
* Custom IP protocols are used (not TCP, UDP, or ICMP)
* Outside-to-inside unsolicited traffic flow is required
* Address traceability is required, typically for security audits
* Easy management and configuration
* Dynamically-allocated state

When traffic traverses the NAT, only the source IP changes. In this example,
inside hosts within ``192.0.2.0/25`` are statically mapped to their matching
address in the pool ``203.0.113.0/25``. The network masks *must* match.
Cisco IOS syntax is below::

  # ip nat inside source static network (inside_local) (inside_global) (pfx_len)
  ip nat inside source static network 192.0.2.0 203.0.113.0 /25

The output from NAT and IP packet debugging shows an identical flow from the
previous section::

  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: s=192.0.2.1->203.0.113.1, d=198.51.100.1
  IP: s=203.0.113.1, d=198.51.100.1, g=203.0.113.253, len 100, forward
  NAT: s=198.51.100.1, d=203.0.113.1->192.0.2.1
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 100, forward

  IP: tableid=0, s=192.0.2.2, d=198.51.100.2, routed via RIB
  NAT: s=192.0.2.2->203.0.113.2, d=198.51.100.2
  IP: s=203.0.113.2, d=198.51.100.2, g=203.0.113.253, len 100, forward
  NAT: s=198.51.100.2, d=203.0.113.2->192.0.2.2
  IP: tableid=0, s=198.51.100.2, d=192.0.2.2, routed via RIB
  IP: s=198.51.100.2, d=192.0.2.2, g=192.0.2.253, len 100, forward

Static One-to-many (1:N) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This technique is used to provide overloaded outside-to-inside access
using TCP or UDP ports. It's particularly useful for reaching devices
typically hidden behind a NAT that need to receive unsolicited traffic.

In these examples, telnet (TCP 23) and SSH (TCP 22) access is needed from
the outside network towards ``192.0.2.1``. To reach this inside local address,
outside clients will target ``203.0.113.1`` using ports 9022 for SSH and
9023 for telnet, respectively. Cisco IOS syntax is below::

  ip nat inside source static tcp 192.0.2.1 22 203.0.113.1 9022
  ip nat inside source static tcp 192.0.2.1 23 203.0.113.1 9023

Generating traffic to test this solution requires more than a simple ``ping``.
From the outside, a user telnets to ``203.0.113.1`` on port 9023 which the NAT
device "forwards" to ``192.0.2.1`` on port 23. That is how this feature earned
the name "port forwarding". The inside local device sees the session coming from
``198.51.100.1``, the outside local address, which in this example is
untranslated::

  R3#telnet 203.0.113.1 9023
  Trying 203.0.113.1, 9023 ... Open
  Username: nick
  Password: nick

  R1#who
      Line       User       Host(s)   Idle       Location
     0 con 0                idle      00:00:47
  * 98 vty 0     nick       idle      00:00:00 198.51.100.1

Enabling NAT (``debug ip nat``) and IP packet (``debug ip packet``) debug
reveals how this technology works. The NAT process is similar but
starts from the outside.

1. Initial packet arrives from ``198.51.100.1`` to ``192.0.2.1`` (not in debug)
2. NAT device translates destination port from 9023 to 23
3. NAT device translates destination from ``203.0.113.1`` to ``192.0.2.1``
4. NAT device perform routing lookup towards ``192.0.2.1``
5. NAT device forwards packet to ``192.0.2.1`` in the inside network
6. Return packet arrives from ``192.0.2.1`` to ``198.51.100.1`` (not in debug)
7. NAT device performs routing lookup towards ``198.51.100.1``
8. NAT device translates source port from 23 to 9023
9. NAT device translates source from ``192.0.2.1`` to ``203.0.113.1``
10. NAT device forwards packet to ``198.51.100.1`` in the outside network

Device output::

  NAT: TCP s=37189, d=9023->23
  NAT: s=198.51.100.1, d=203.0.113.1->192.0.2.1
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 42, forward
  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: TCP s=23->9023, d=37189
  NAT: s=192.0.2.1->203.0.113.1, d=198.51.100.1
  IP: s=203.0.113.1, d=198.51.100.1, g=203.0.113.253, len 42, forward

The next example is almost identical except uses SSH. Use this opportunity
to test your understanding by following the debug output and reciting
the NAT order of operations::

  R3#ssh -l nick -p 9022 203.0.113.1

  Password: nick

  R1#who
      Line       User       Host(s)              Idle       Location
     0 con 0                idle                 00:02:23
  * 98 vty 0     nick       idle                 00:00:00 198.51.100.1

    Interface    User               Mode         Idle     Peer Address

  NAT: TCP s=20534, d=9022->22
  NAT: s=198.51.100.1, d=203.0.113.1->192.0.2.1 [21542]
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 40, forward
  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: TCP s=22->9022, d=20534
  NAT: s=192.0.2.1->203.0.113.1, d=198.51.100.1
  IP: s=203.0.113.1, d=198.51.100.1, g=203.0.113.253, len 268, forward
  
Dynamic One-to-many (1:N) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This type of NAT is the most commonly deployed. Almost every consumer Internet
connection has a LAN network for wired/wireless access. All hosts on this
segment are translated to a single source IP address by using layer-4 source
port overloading. The changed source port serves as the demultiplexer to
translate return traffic back to the proper source IP address. 

This solution is also called NAT overload, Port Address Translation (PAT), or
Network Address/Port Translation (NAPT). The solution can consume a NAT pool
(range) but can reuse inside global addresses in the pool across many inside
local addresses::

  # ACL defines the inside local addresses (pre NAT source)
  ip access-list standard ACL_INSIDE_SOURCES
   permit 192.0.2.0 0.0.0.127

  # Pool defines the inside global addresses (post NAT source)
  ip nat pool POOL_203 203.0.113.12 203.0.113.31 prefix-length 27

  # Binds the inside local list to the inside global pool for translation
  # Addresses allocated from the pool can be re-used
  ip nat inside source list ACL_INSIDE_SOURCES pool POOL_203 overload

In order to see this solution, ``debug ip nat detailed`` is used to more
explicitly show the inside and outside packets and their layer-4 ports. The
first example uses telnet from ``192.0.2.1`` to ``198.51.100.1``.

1. Initial packet arrives from ``192.0.2.1`` to ``198.51.100.1`` (not in debug)
2. NAT device performs routing lookup towards ``198.51.100.1``
3. NAT device identifies inside translation using source port 57186.
4. NAT device translates source from ``192.0.2.1`` to ``203.0.113.12``
5. NAT device forwards packet to ``198.51.100.1`` in the outside network
6. Return packet arrives from ``198.51.100.42`` to ``203.0.113.1`` (not in debug)
7. NAT device identifies outside translation using destination port 57186.
8. NAT device translates destination from ``203.0.113.12`` to ``192.0.2.1``
9. NAT device perform routing lookup towards ``192.0.2.1``
10. NAT device forwards packet to ``192.0.2.1`` in the inside network

Device output below explains how this works::

  IP: tableid=0, s=192.0.2.1, d=198.51.100.1, routed via RIB
  NAT: i: tcp (192.0.2.1, 57186) -> (198.51.100.1, 23)
  NAT: s=192.0.2.1->203.0.113.12, d=198.51.100.1
  IP: s=203.0.113.12, d=198.51.100.1, g=203.0.113.253, len 42, forward
  NAT: o: tcp (198.51.100.1, 23) -> (203.0.113.12, 57186)
  NAT: s=198.51.100.1, d=203.0.113.12->192.0.2.1
  IP: tableid=0, s=198.51.100.1, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.1, g=192.0.2.253, len 42, forward

The power of the solution is illustrated in the output below. A new source,
``192.0.2.2`` also initiates a telnet session to ``198.51.100.1`` and is
translated to the same inside global address ``203.0.113.12`` except has
a source port of 55943. Try to follow the order of operations::

  IP: tableid=0, s=192.0.2.2, d=198.51.100.1, routed via RIB
  NAT: i: tcp (192.0.2.2, 55943) -> (198.51.100.1, 23)
  NAT: s=192.0.2.2->203.0.113.12, d=198.51.100.1
  IP: s=203.0.113.12, d=198.51.100.1, g=203.0.113.253, len 42, forward
  NAT: o: tcp (198.51.100.1, 23) -> (203.0.113.12, 55943)
  NAT: s=198.51.100.1, d=203.0.113.12->192.0.2.2
  IP: tableid=0, s=198.51.100.1, d=192.0.2.2, routed via RIB
  IP: s=198.51.100.1, d=192.0.2.2, g=192.0.2.253, len 42, forward

The solution, while very widely used, has many drawbacks:

* Many hosts use a common IP address; hard to trace
* Applications that require source ports to remain unchanged may not work.
  The NAT would have to retain source ports, which assumes inside local
  devices never use the same source port for inside-to-outside flows.
* It only works TCP and UDP traditionally, but most implementations also
  support ICMP. Protocols like GRE, IPv6-in-IPv4, and L2TP do not work.

Twice NAT
^^^^^^^^^
This NAT technique is by far the most complex. It is generally limited to
in environments where there are overlapping IPs between two hosts that must
communicate directly. This can occur between RFC 1918 addressing when
organizations undergo mergers and acquisitions.

In this example, an inside host ``192.0.2.1`` needs to communicate to an
outside host ``198.51.100.1``. Both of these problems exist:

1. There is already a host with IP ``198.51.100.1`` on the inside network
2. There is already a host with IP ``192.0.2.1`` on the outside network

To make this work, each host must see the other as some other address.
That is, both the inside source and outside source must be concurrently
translated, hence the name "twice NAT". This should not be confused with
"double NAT" which is discussed in the CGN section.

Twice NAT is where all four of the NAT address types are different:

* inside local: ``192.0.2.1``
* inside global: ``203.0.113.1``
* outside global: ``198.51.100.1``
* outside local: ``192.0.2.99``

Cisco IOS example syntax::

  # ip nat inside source static (inside_local) (inside_global)
  ip nat inside source static 192.0.2.1 203.0.113.1

  # ip nat outside source static (outside_global) (outside_local)
  ip nat outside source static 198.51.100.1 192.0.2.99 add-route

The order of operations is similar to previous inside source NAT examples,
except that following every source translation is a destination translation.

1. Initial packet arrives from ``192.0.2.1`` to ``192.0.2.99`` (not in debug)
2. NAT device performs routing lookup towards ``192.0.2.99``
3. NAT device translates source from ``192.0.2.1`` to ``203.0.113.1``
4. NAT device translates destination from ``192.0.2.99`` to ``198.51.100.1``
5. NAT device forwards packet to ``198.51.100.1`` in the outside network
6. Return packet arrives from ``198.51.100.1`` to ``203.0.113.1`` (not in debug)
7. NAT device translates source from ``198.51.100.1`` to ``192.0.2.99``
8. NAT device translates destination from ``203.0.113.1`` to ``192.0.2.1``
9. NAT device perform routing lookup towards ``192.0.2.1``
10. NAT device forwards packet to ``192.0.2.1`` in the inside network

Device output showing the order of operations is below::

  IP: tableid=0, s=192.0.2.1, d=192.0.2.99, routed via RIB
  NAT: s=192.0.2.1->203.0.113.1, d=192.0.2.99 [40]
  NAT: s=203.0.113.1, d=192.0.2.99->198.51.100.1 [40]
  IP: s=203.0.113.1, d=198.51.100.1, g=203.0.113.253, len 100, forward
  NAT*: s=198.51.100.1->192.0.2.99, d=203.0.113.1 [40]
  NAT*: s=192.0.2.99, d=203.0.113.1->192.0.2.1 [40]
  IP: tableid=0, s=192.0.2.99, d=192.0.2.1, routed via RIB
  IP: s=192.0.2.99, d=192.0.2.1, g=192.0.2.253, len 100, forward

NAT as a crude load balancer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
This use case is uncommonly used but does, at a basic level, represent how
a server load balancer might work. It combines the logic of port forwarding
(unsolicited outside-to-inside access) with a rotary NAT pool to create
a dynamic solution whereby external clients can access internal hosts in
round-robin fashion, typically for load balancing.

Cisco IOS syntax example::

  # Define a list of virtual IPs for outside-to-inside access
  ip access-list standard ACL_VIRTUAL_IP
   permit 203.0.113.99

  # Define the inside local "servers" in the pool
  ip nat pool POOL_192 192.0.2.1 192.0.2.4 prefix-length 9 type rotary

  # Bind the virtual IP list to the server pool
  ip nat inside destination list ACL_VIRTUAL_IP pool POOL_192

Each time the outside host with IP ``198.51.100.1`` telnets to the
virtual IP address used to represent the server pool of ``203.0.113.99``,
the NAT device selects a different inside local address for the destination
of the connection. The range of inside IP addresses goes from ``192.0.2.1``
to ``192.0.2.4``, which are actual server IP addresses behind the NAT.
The debug is shown below with RIB/packet forwarding output
excluded for clarity as it reveals nothing new. Each block of output
is from a separate telnet session, and represents a single keystroke::
  
  # Choose server 192.0.2.1
  NAT*: s=198.51.100.1, d=203.0.113.99->192.0.2.1
  NAT*: s=192.0.2.1->203.0.113.99, d=198.51.100.1

  # Choose server 192.0.2.2
  NAT*: s=198.51.100.1, d=203.0.113.99->192.0.2.2
  NAT*: s=192.0.2.2->203.0.113.99, d=198.51.100.1

  # Choose server 192.0.2.3
  NAT*: s=198.51.100.1, d=203.0.113.99->192.0.2.3
  NAT*: s=192.0.2.3->203.0.113.99, d=198.51.100.1

  # Choose server 192.0.2.4
  NAT*: s=198.51.100.1, d=203.0.113.99->192.0.2.4
  NAT*: s=192.0.2.4->203.0.113.99, d=198.51.100.1

Carrier Grade NAT (CGN)
^^^^^^^^^^^^^^^^^^^^^^^
TODO: LSN, double NAT

NAT as a security tool
^^^^^^^^^^^^^^^^^^^^^^
TODO: topology hiding, minor security advantage

The true cost of NAT
^^^^^^^^^^^^^^^^^^^^
TODO: CGN logging opex, hardware capex, common abuses, ALG
