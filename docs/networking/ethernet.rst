========
Ethernet
========

Link Aggregation
----------------
Link aggregation is a general term to describe using multiple Ethernet
interfaces simultaneously. There are several terms for this, including bonding,
teaming, EtherChannel, and more. However, there are ongoing debates as to what
each term means specifically, hence the reason there are many different terms
to describe the same overall concept. Link aggregation has two primary goals:
increased throughput and increased redundancy.

From a network infrastructure context, link aggregation normally means to
combine two or more physical Ethernet interfaces with identical properties
(such as link speed and duplex) into a single logical interface. For example,
two physical 10 Gbps Ethernet links can be combined into a single 20 Gbps
logical link. However, the data is not actually transmitted at 20 Gbps.

The most common analogy for link aggregation is adding more lanes to a highway,
but the speed limit stays the same. In other words, with multiple aggregated
10 Gbps interfaces, a data transfer in a single session will not go faster than
10 Gbps, but multiple sessions can be distributed over each of the links, which
enables an aggregate throughput of more than an individual link.

Why is link aggregation needed?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Network switches build internal tables to correspond MAC addresses to
interfaces so network traffic can be appropriately forwarded. Within the switch,
this is referred to as the Content Addressable Memory (CAM) table. When you
connect a device such as a PC or a server to a port on a switch, the switch
sees the MAC address of the device's network interface and records in the CAM
table that it is associated with that particular port. When devices communicate
with each other, the switch knows which interfaces to forward the traffic
between based on the CAM table.

Individual MAC addresses can be associated to only a single interface in the
CAM table, though each interface can have multiple MAC addresses associated. A
single IP address is typically associated with a single MAC address (with some
exceptions). The Address Resolution Protocol (ARP) runs on both the host and
the host's default gateway to map IP addresses to MAC addresses. When link
aggregation is used, a MAC address is assigned to the aggregated logical
interface. When an IP address is configured on the logical link, ARP resolves
the IP address to the MAC address of the virtual link. This allows the multiple
physical interfaces to be used simultaneously with the single IP address.

How is link aggregation different than NIC teaming?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Link aggregation requires coordination between the network infrastructure and
the device containing multiple physical links with either static configuration
or a negotiation protocol such as Link Aggregation Control Protocol (LACP).
NIC teaming does not require coordination with the network infrastructure and
is entirely host-dependent. With link aggregation, both the host and the
network infrastructure share the same view of the local network. With NIC
teaming, the network is unaware of the configuration on the host.

For example, VMware's ESXi operating system supports both link aggregation and
NIC teaming. When link aggregation with LACP is configured, both the network
and ESXi host share the same view of the network with the logical link. By
default, ESXi uses NIC teaming. Virtual machines have virtual network
interfaces that each have their own associated MAC addresses. When the VM is
powered on, the virtual network interfaces are associated with a network
interface on the ESXi host. When LACP is used, the VM MAC address becomes
associated with the LACP logical link. When the default NIC teaming is used,
the VM MAC address becomes associated with one of the ESXi host's physical
network interfaces. When a second VM is powered on, its VM MAC address is
associated with a different physical network interface on the ESXi host. This
facilitates rudimentary network load distribution across multiple physical
interfaces on the ESXi host without involving the network infrastructure.

Both LACP and NIC teaming support "active/active" and "active/standby"
configurations. In "active/active" mode, all links are used simultaneously. In
"active/standby" mode, one or more physical links are intentionally disabled.
If one of the active links fails, the standby link becomes active and takes
over for the failed link.

Why would I use link aggregation instead of NIC teaming?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Using VMware ESXi as an example, NIC teaming maps each virtual network
interface (vNIC) onto a single host physical network interface. The mapping
does not change unless the VM is powered off or migrated to another host, or
if the physical link on the host goes down. This means network traffic to and
from the VM's vNIC will always traverse the same physical interface and is
limited to the speed of that interface.

With link aggregation using LACP, the same VM vNIC is mapped to the ESXi
host's logical aggregated interface. The same rules still apply where a single
network session (also known as a flow) is still limited to the speed of an
individual physical link, but multiple flows can take advantage of the
multiple links simultaneously, thereby increasing the overall throughput.

For example, with NIC teaming, if a VM is acting as a web server, it will use
the same physical link when serving content to multiple clients. With link
aggregation, the same VM will use multiple physical interfaces across the
single logical interface to serve content to clients.

The choice of using link aggregation or NIC teaming is partly dependent on the
network demands of your workload. If redundancy is your primary concern without
regard for throughput, such as for workloads that do not generate large volumes
of network traffic, NIC teaming may be the best and simplest option. However,
workloads that deal with high volumes of network traffic are typically served
better by using link aggregation, which provides both redundancy and higher
overall network throughput across multiple sessions.

What happens if I configure link aggregation on the host, but not the network?
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Unlike NIC teaming, link aggregation requires coordination with the network
infrastructure. The two most common methods are static configuration, and LACP
(originally IEEE 802.3ad, now part of IEEE 802.1AX). With LACP, the host and
the network switch communicate with each other to make sure each physical link
is part of the same link bundle (otherwise known as a Link Aggregation Group
or LAG). Any links that are determined to not be a part of the bundle are
disabled to prevent forwarding loops and other issues.

For example, assume two different LAGs are configured on a switch and LACP is
used. Two different servers are connected with multiple links, one server per
LAG. If the links are accidentally misconnected so that each server connected
to both LAGs, LACP will detect this and disable the links that do not belong
to the proper group. Static configuration has no way to detect this problem,
and is therefore not recommended unless both the server and the network switch
do not support LACP (which is extremely rare).

Remember that in a network switch, a single MAC address must correspond with a
single entry in its CAM table, which represents the interface the MAC address
is associated with. If the server is configured to use link aggregation but
the network is not, MAC addresses associated with the server appear on
multiple switch interfaces simultaneously. Since a MAC address can only be on
a single switch interface, the MAC address is disassociated from one interface
and re-associated with another as the MAC address is seen from the connected
server.

This disassociation and re-association happens over and over again between
interfaces on the switch. This is known as "MAC flapping". Different network
equipment handles this situation differently. Typically, network interfaces
associated with MAC flapping are disabled, either permanently or for a short
period of time. Log messages on the switch are also typically generated.

Here is an example of the error message generated on a switch running Cisco's
NX-OS: ::

  switch1 %L2FM-2-L2FM_MAC_FLAP_DISABLE_LEARN_N3K: Loops detected in the
  network for mac 1234.5678.90ab among ports Eth1/1 and Eth1/2 vlan 111 -
  Disabling dynamic learning notifications for a period between 120 and 240
  seconds on vlan 111

  switch1 %L2FM-2-L2FM_MAC_FLAP_RE_ENABLE_LEARN_N3K: Re-enabling dynamic
  learning on vlan 111

In this case, Cisco's NX-OS handles the issue by disabling dynamic MAC
learning for a period of time for the entire VLAN. This means that during this
"quiet" period, no MAC address changes will be registered for that VLAN. If a
new device comes online in the VLAN, or an existing device changes ports, the
change will not be registered in the switch and the device will not be able to
communicate with the network until MAC learning is re-enabled. This will
happen over and over again until the issue is corrected. The issue can be
corrected by shutting down the misconfigured links or correctly configuring
link aggregation between the switch and the server.

What is MC-LAG?
^^^^^^^^^^^^^^^

Traditional link aggregation involves multiple connections to a single switch.
Multi-Chassis Link Aggregation Groups aim to further increase redundancy
levels by connecting a single device, such as a server, to multiple switches
while still presenting a single logical interface to the server. This
introduces device-level redundancy along with link-level redundancy. Currently,
all MC-LAG solutions are proprietary to the networking vendor, and require both
switches to be running the same network operating system.

For example, Cisco's NX-OS uses an MC-LAG technology called "Virtual Port
Channel", or vPC. Two switches running NX-OS are configured to recognize each
other and present a single unified LACP-based LAG to the downstream device,
such as a server. The server believes it is connected to a single upstream
switch. The two switches coordinate with each other to handle the redundancy
and prevent loops and MAC flapping.
