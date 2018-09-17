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

* **inside**: The network *behind* the NAT point, often where source
  addresses are translated from private to public.
* **outside**: The network *in front of* the NAT point. Often packets in
  this network hae publicly routable source IP addresses.
* **state**: The information stored in the NAT device about each translation.
  NAT state is typically stored in a table containing pre and post NAT
  addressing, and sometimes layer-4 port/protocol information as well.
* **pool**: A range of addresses which can be used as outputs from the NAT
  process. Addresses in a pool are dynamic resources which are allocated
  on-demand when hosts need to traverse a NAT and require different IPs.
* **ALG**: Application Level Gateway is a form of payload inspection that
  looks for embedded IP addresses inside the packet's payload, which
  traditional NAT devices cannot see. ALGs are NAT enhancements on a
  per-application basis (yuck) to translate these embedded addresses.
  Applications such as active FTP and SIP require ALGs.

In Cisco-speak, some additional terms are defined. These seem confusing at
first, but are a blessing in disguise because they are unambiguous.

* **Inside local:** IP on the *inside* network from the perspective of other
  hosts on the *inside* network. This is often a private RFC 1918 IP
  address and is seen by other hosts on the inside as such.
* **Inside global:** IP on the *inside* network from the perspective of
  hosts on the *outside* network. This is often a publicly routable
  address which represents the inside host to outside hosts.
* **Outside global:** IP on the *outside* network from the perspective of other
  hosts on the *outside* network. This is often a publicly routable
  address and is seen by other hosts on the outside as such.
* **Outside local**: IP on the *outside* network from the perspective of
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
* This document *describes* NAT, but does not *prescribe* NAT. NAT designs are
  almost never the best overall solutions and should be considered short-term,
  tactical options. Software should avoid relying on it by using duplicate,
  hardcoded IPs, then relying on NAT later to fix the blunder.
* Mastery of the NAT order of operations is required for any developer
  building an application that may traverse a NAT device. This affects which
  addresses are used when, impacts DNS, impacts packet payloads
  (ie, embedded IPs), and more.

  1. Packet arrives on inside
  2. Router performs route lookup to destination
  3. Router translates address
  4. Reply arrives on outside
  5. Route translates address back
  6. Router performs route lookup to destination

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

Static Many-to-one (N:1) Inside Source NAT
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
  
Dynamic Many-to-one (N:1) Inside Source NAT
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
This NAT technique does not introduce any new technology as it usually
refers to simply chaining PAT solutions (dynamic inside source NAT) in
series to create an aggregated, more overloaded design. Consider the home
network of a typical consumer that has perhaps 5 IP-enabled devices on the
network concurrently. Each device may have 50 connections for a total of 250
flows. A single IP address could theoretically support more than 65,000 flows
using PAT given the expanse of the layer-4 port range. Aggregating flows at
every higher points in a hierarchical NAT design helps achieve greater
NAT efficiency/density. Thus, given ~250 flows per home, a single CGN could
perform NAT aggregation services for ~260 homes to result in ~65,000
ports utilized from a single address.

CGN is also known as Large Scale NAT (LSN), LSN444, NAT444, hierarchical NAT,
and double NAT. Do not confuse CGN with "twice NAT" which is typically a
single NAT point where the source *and* destination are both translated.

Cisco IOS sample syntax on the home router (first NAT point)::

  # ACL defines the inside local addresses (pre NAT source)
  ip access-list standard ACL_INSIDE_SOURCES
   permit 192.0.2.0 0.0.0.127

  # Use the single outside IP (often a DHCP address) to translate inside hosts
  ip nat inside source list ACL_INSIDE_SOURCES interface FastEthernet0/1 overload

Cisco IOS sample syntax on the CGN router (second NAT point)::

  # Contains the inside global addreses (CGN range) from the home routers
  ip access-list standard ACL_CONSUMER_ROUTERS
   permit 100.64.0.0 0.63.255.255

  # Create NAT pool, typically a small number of public IPs for the CGN domain
  ip nat pool POOL_CGN 203.0.113.0 203.0.113.3 prefix-length 30

  # Bind the home routers using CGN 100.64.0.0/10 space to the CGN pool
  ip nat inside source list ACL_CONSUMER_ROUTERS pool POOL_CGN overload

Given that no new technology is introduced, the debug below can be summarized
in 4 main steps. The debugs are broken up to help visualize the NAT actions
across multiple devices along the packet's journey upstream and downstream.

1. Inside-to-outside NAT at home router
2. Inside-to-outside NAT at CGN router
3. Outside-to-inside NAT at CGN router
4. Outside-to-inside NAT at home router

Device output::

  # Inside-to-outside NAT at home router
  IP: tableid=0, s=192.0.2.1, d=198.51.100.4, routed via RIB
  NAT: s=192.0.2.1->100.64.23.254, d=198.51.100.4
  IP: s=100.64.23.254, d=198.51.100.4, g=100.64.23.253, len 100, forward

  # Inside-to-outside NAT at CGN router
  IP: tableid=0, s=100.64.23.254, d=198.51.100.4, routed via RIB
  NAT: s=100.64.23.254->203.0.113.2, d=198.51.100.4
  IP: s=203.0.113.2, d=198.51.100.4, g=203.0.113.254, len 100, forward

  # Outside-to-inside NAT at CGN router
  NAT*: s=198.51.100.4, d=203.0.113.2->100.64.23.254
  IP: tableid=0, s=198.51.100.4, d=100.64.23.254, routed via RIB
  IP: s=198.51.100.4, d=100.64.23.254, g=100.64.23.254, len 100, forward

  # Outside-to-inside NAT at home router
  NAT*: s=198.51.100.4, d=100.64.23.254->192.0.2.1
  IP: tableid=0, s=198.51.100.4, d=192.0.2.1, routed via RIB
  IP: s=198.51.100.4, d=192.0.2.1, g=192.0.2.253, len 100, forward

Like most "quick fix" solutions, CGN has serious drawbacks:

* Massive capital investment for specialized hardware to perform high-density
  NAT with good performance and reliability.
* Burdensome regulatory requirements on NAT logging. If ~260 households all
  share a single public IP, criminal activity tracking becomes more difficult.
  Carriers are often required to retain NAT logging, along with consumer router
  IP addressing bindings, to identify where malicious activity originates. This
  comes with additional capital and operating expense.
* ALG support is already hard with NAT. With CGN, almost impossible.
* No possibility of outside-to-inside traffic, even with port forwarding,
  for customers.
* No possibility of peer-to-peer applications working.
* Restricted ports per consumer (250 in the theoretical example above)

In closing, developers should build applications that can use IPv6, totally
obviating the complex and costly workarounds need to get them working across
CGN in many cases.

NAT as a security tool
^^^^^^^^^^^^^^^^^^^^^^
This is a topic of much debate, and there are two sides to the coin.

NAT provides the following security advantages:

  * No reachability: In one-to-many NAT scenarios where unsolicited
    outside-to-inside flows are technically impossible, the inside hosts
    are insulted from direct external attacks.
  * Obfuscation: The original IP address of the inside host is never
    revealed to the outside world, making targeted attacks more difficult.
  * Automatic firewalling: As a stateful device, only outside-to-inside
    flows already in the state table are allowed, and the inside-to-outside
    NAT state is created based on an ACL, much like a firewall.

Many of these security advantages are easily defeated:

  * Relatively simple attacks like phishing and social engineering cause
    clients to initiate connections to the outside. Such attacks are
    easy to stage and more effective than outside-in frontal assaults.
  * Most applications don't use their IP addresses as identification. A web
    application might have usernames or a digital certificate. The IP address
    itself is mostly irrelevant and not worth protecting. It's entirely
    irrelevant when clients receive IP addresses dynamically, eg via DHCP.
  * A NAT device drops outside-in flows due to lack of state, but does not
    provide any protection against spoofed, replayed, or otherwise bogus
    packets that piggy-back on existing flows. Security appliances do.
    The only similarity between NATs and firewalls is that they maintain
    state. They should not be categorized in any other way.

In closing, do not rely on NAT as a real security technology. It's roughly
equivalent to closing your blinds in your home. It adds marginal security
value but there are clearly better alternatives. Like blinds on windows,
NAT may conceal and slow down some attacks, but should never be confused for
a legitimate security component in the design.

.. sectionauthor:: Nick Russo <njrusmc@gmail.com>
