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
TODO: IP NAT with pool

Static One-to-many (1:N) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
TODO: Port forwarding

Dynamic One-to-many (1:N) Inside Source NAT
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
TODO: NAT overload, NAPT, PAT

Twice NAT
^^^^^^^^^
TODO: Translate source and destination concurrently

Carrier Grade NAT (CGN)
^^^^^^^^^^^^^^^^^^^^^^^
TODO: LSN, double NAT

NAT as a crude load balancer
^^^^^^^^^^^^^^^^^^^^^^^^^^^^
TODO: inside destination NAT

NAT as a security tool
^^^^^^^^^^^^^^^^^^^^^^
TODO: topology hiding, minor security advantage

The true cost of NAT
^^^^^^^^^^^^^^^^^^^^
TODO: CGN logging opex, hardware capex, common abuses
