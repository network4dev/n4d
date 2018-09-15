=====
Azure
=====

Microsoft Azure is a relative newcomer in cloud computing compared to AWS. Over 
the past couple of years Microsoft has done some incredible work with the Azure 
platform and open source community to become one of the leading cloud providers.

The services they offer include compute, storage, database, serverless, CDN 
and much more. The purpose of this section is to describe the networking
services in Azure and an overview of Azure networking.


-----------------------------
Networking in Azure Overview
-----------------------------
Networking in public cloud is quite different from an on-premises network.
In order to build a multi-tenant, scalable, fault tolerant and high speed
network, Azure has designed the network quite differently than a
"traditional" network. The most notable difference is:

* No support for multicast
* No support for broadcast

What this means is that routing protocols using multicast are not
supported without some form of tunneling but more importantly this means
that because broadcast is not supported, ARP requests are not flooded
in the network. Without ARP, how can a host learn the MAC address of
another host? This requires some magic. Which is performed by the
hypervisor. When the hypervisor sees an ARP request, it will proxy
the ARP response back to the host. More about the specifics of networking
in Azure later.

-------------------
Azure Fundamentals
-------------------

^^^^^^^^^^^^^^^^^^^^
Geopolitical Region 
^^^^^^^^^^^^^^^^^^^^
A Geopolitical region is a collection of Azure regions which can be reached 
from a standard ExpressRoute.

https://docs.microsoft.com/en-us/azure/expressroute/expressroute-locations has more details around this.

^^^^^^^
Regions & Availability Zones
^^^^^^^
A Region in Azure is one or several datacenters in the same
geographical area sharing infrastructure such as power and internet transports.

Within a region there are multiple Availability Zones.
Each Availability Zone is a separate physical location, for example a separate data centre room, or another building with very low latency to the other availability zones within the region.

.. image:: https://azurecomcdn.azureedge.net/cvt-8efd50c617ae2a512e0e084105e16e030ebe42305c0e7bc7ece951fb57e33112/images/page/global-infrastructure/regions/understand-regional-architecture.png

https://azure.microsoft.com/en-gb/global-infrastructure/regions/

^^^^^^^^^^^^^^^^^^^^
Resource Group (RG)
^^^^^^^^^^^^^^^^^^^^
A resource group is a logical grouping of infrastructure resources.
For example a Production RG may include:

* Production VNet
* Storage account(s) related to VMs
* VM Servers
* Azure Traffic Manager DNZ zones

^^^^
VNet
^^^^

A VNet is an IP Prefix allocated for use in Azure.

You can have multiple VNets within a subscription/resource group.

VNets are isolated instances by default, if you have multiple VNets within a region
or globally they will be unable to communicate with each other. 

VNet Peering, VPNs, or ExpressRoute can allow this communication.

Example of VNet assignments might be:

+--------------+-----------------------+
| VNet         | Assignment            |
+--------------+-----------------------+
| 10.0.0.0/16  | Western Europe (Prod) |
+--------------+-----------------------+
| 10.0.16.0/16 | North Europe (DR)     |
+--------------+-----------------------+

^^^^^^
Subnet
^^^^^^
A subnet is an IP Prefix allocation within a VNet.
Multiple Subnets are generally assigned within a VNet. Depending on the application deployment model
you might want to assign a VNet for Management, Apps, Databases, etc...

.. note:: This is just an example and not a recommended subnet layout.

+--------------+----------------+
| Subnet       | Usage          |
+--------------+----------------+
| 10.0.0.0/24  | GatewaySubnet  |
+--------------+----------------+
| 10.0.1.0/24  | Management     |
+--------------+----------------+
| 10.0.2.0/24  | Apps           |
+--------------+----------------+
| 10.0.3.0/24  | Database       |
+--------------+----------------+

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Network Security Group (NSG)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
A NSG is a stateful firewall in Azure. They can be associated either with a specific VM, or with an entire subnet.
See https://docs.microsoft.com/en-us/azure/virtual-network/security-overview for more information.

^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Network Virtual Appliance (NVA)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
A Network Virtual Appliance (NVA) is the name for using a traditional network vendor VM in Azure. 

For example you can deploy VMs from vendors such as Juniper VSRX firewalls, Palo Alto firewalls, Cisco CSR routers and integrate them into your standard networking toolset.

There is nothing special about these VMs, many people deploy Linux VMs to manage routing and  firewalling.

^^^^^^^^^^^^^^^^^^^^^^^^^^
User Defined Routes (UDR)
^^^^^^^^^^^^^^^^^^^^^^^^^^
UDR is a routing table within Azure. It allows static routes to be defined and system
Routes overwritten.
UDRs are assigned at a subnet level, and you can assign the same UDR to multiple subnets.
Generally you wouldn’t use a UDR unless you’re using a Network Virtual Appliance (NVA)



.. sectionauthor:: Daniel Rowe <n4d@danrowe.email>