# Virtual Cloud Networks

- [Virtual Cloud Networks](#virtual-cloud-networks)
  - [CIDR notation](#cidr-notation)
  - [VCN](#vcn)
    - [Hands on with VCN](#hands-on-with-vcn)
    - [Subnets](#subnets)
      - [vNICs](#vnics)
      - [Private IP](#private-ip)
      - [Public IP](#public-ip)
    - [Internet Gateway](#internet-gateway)
    - [Route Table](#route-table)
    - [NAT Gateway](#nat-gateway)
    - [Service Gateway](#service-gateway)
      - [CIDR Labels](#cidr-labels)
    - [Dynamic Routing Gateway](#dynamic-routing-gateway)
    - [VCN Peering](#vcn-peering)
    - [Local Peering Gateway](#local-peering-gateway)
    - [Remote Peering](#remote-peering)
    - [Transit Routing](#transit-routing)
    - [Security Lists](#security-lists)
    - [Default Components for the VCN](#default-components-for-the-vcn)
    - [Internal DNS](#internal-dns)
  - [Edge Security](#edge-security)
  - [Questions](#questions)

## CIDR notation

Recollect the basics :

* An IP address is 32 bits. 
  * The 32 bits are divided in to four 8-bit octets.
  * An Ip address in binary looks like this :
    * `00000000.00000000.00000000.00000000` to
    * `11111111.11111111.11111111.11111111`
  * Since thats obviously inconvinient, we represent each of those 8-bit segments (octets) in decimal.
  * `00000000` => `0` and `11111111` => `255` in decimal form.
  * So in decimal form we can represent the same range as `0.0.0.0`-`255.255.255.255`
* CIDR represents chunk of contigous IP addresses => CIDR Block
* The notation simply denotes an IP addres with the number of bits that are fixed.
* The remaining number of bits gives the possibilities for IP addresses in that block = the size of the CIDR block
  * `10.0.0.0/30` ( _Remember that there are only 32 bits in an IPv4 address_ )
    * This CIDR says 30 bits are fixed, so only 2 bits can change => we have 2<sup>2</sup> = 4 possible addresses
    * They range from `10.0.0.0`-`10.0.0.3`
    * So a CIDR Block `10.0.0.4/30` can represent another non-overlapping CIDR Block -> `10.0.0.4`-`10.0.0.7` (4 IPs)
* So a `10.0.0.0/24` CIDR block has 24 bits reserved, and we have 32-24 = 8 bits available to us for allocation
  * This gives us 2<sup>8</sup> = 256 addresses, from  `10.0.0.0`-`10.0.0.255`
* Another example `10.0.0.0/20`
  * 20 bits are blocked, leaving 12 bits for the addresses, or 2<sup>12</sup> = 4096 addresses.
  * Of the 20 bits blocked => first two octets, and the first 4 bits of the 3rd octet, leaving 4 bits in the 3rd octect
    * **00001010.00000000.0000**_0000.00000000_
  * The third octect can range from `0000`-`1111` => 0-15 (with 4 bits we can have 2<sup>4</sup>=16 values)
  * So the 4096 addresses will be `10.0.0.255`, `10.0.1.255`, ... `10.0.15.255`

## VCN

VCNs represent distinct networks, in the tenancy with their own CIDR blocks.

* They are regional. Two regions, require two VCNs, one per region (minimum)
* CIDR ranges can be /16 biggest network, to /30 smallest network. (10.0.0.0/8, 172.16/12 are in RFC, but not supprted by VCN)
* VCN reserves the first **TWO** IP addresses and the last IP address in each CIDR block. 
  * Normally the first address .0 is used by the network itself, and the last one .255 is used for broadcast.
  * OCI in addition to this reserves one more, the second IP in the block.
  * So a /30 block will yeild **one** usable address, instead of 4.

### Hands on with VCN

To create a VCN, the CIDR Block and the comaprtment ID are mandatory.

```bash
oci network vcn create \
--cidr-block 192.168.0.0/24 \
--compartment-id <ocid> \
--dns-label certprep
```

The newly created VCN will have the default resources created and the command returns the OCIDs for the resources created.

```json
{
  "data": {
    "cidr-block": "192.168.0.0/24",
    "compartment-id": "ocid1.compartment.oc1..aaaaaaaa4vxl6yyvfcumwutejntiu3tzcwacbpgdqndh3kct5i65ahvz7oma",
    "default-dhcp-options-id": "ocid1.dhcpoptions.oc1.phx.aaaaaaaap46hsz4xtxpjl7dktkuvdljbmzo3u6bcuvebmtvafh3imffwxbga",
    "default-route-table-id": "ocid1.routetable.oc1.phx.aaaaaaaac4v7lt47xqp63fht7jjd5ima7jeomafsek4p4rhjdhbkomrlpalq",
    "default-security-list-id": "ocid1.securitylist.oc1.phx.aaaaaaaaj6temm3uifjmy2r6l5n7lkr36bjvxrep7lulbj5grn4odnlxxgtq",
    "defined-tags": {},
    "display-name": "vcn20190325075458",
    "dns-label": "certprep",
    "freeform-tags": {},
    "id": "ocid1.vcn.oc1.phx.aaaaaaaao5waj4btfvwlvx6pfhb4ntvydt6msacvoxt2dz5uy4426kimjqaa",
    "lifecycle-state": "AVAILABLE",
    "time-created": "2019-03-25T07:54:58.315000+00:00",
    "vcn-domain-name": "certprep.oraclevcn.com"
  },
  "etag": "4f09a296"
}
```

### Subnets

Subnets in an VCN is the logical equivalent of an actual subnet.
They divide the IP namespace of the VCN in to distinct groups that can be managed indepdently. eg: a DB subnet can be isolated from a web-tier subnet.

* Are an AD level construct, and can be **Public** or **Private**
* **Public** subnets allow the use of public IP addresses on vNICs inside the subnet.
* As subnets are AD specific, the VCN CIDR block is divided in to multiple subnets at the AD level.
  * The subnet address range should be within the VCN block
  * Subnet addressing ranges cannot overlap
  * Subnet can have one Route table and upto 5 Security Lists

Quick creating a subnet with the console will create the subnet and the default resources like the seclist, route table and dhcp option,
and additionally will create 3 subnets distributed in the 3 ADs, and choose the CIDR blocks for the VCN (will be a /16 network) as well as the subnets.

#### vNICs

A vNIC is a networking service in OCI that enables the communication from an instance to the underlying physical NIC card.
  * X5 hardware has 2 NICs, but only one is active
  * X7 hardware has 2 NICs of 25 Gbps each and both are active
  * Primary vNIC
    * Every instance has a primary vNIC thats created and configured at lauch time
    * Primary vNICs cannot be removed from an isntance.
  * Secondary vNIC
    * This can be in the same subnet, a different one, a whole different VCN. 
    * It **must be in the same AD**
      * It cannot be in a subnet thats in a different availability domain
  * The number of vNICs that can be attached to an instance depends on the shape.
  * A vNIC must always be attached to an isntance.
    * Detacing the vNIC deletes it
    * They are detached and deleted when an instance terminates
  * The bandwidth available to an instance does not depend on the number of vNICs. That just depends on the instance shape. 

#### Private IP

* Every instance has a private IP associated with it.
* A private IP can optionally have a public IP associated with it if its in a public subnet.
* Every Instance has at least a primary. Can have more.
  * The vNICs on an instance could be connected to multiple subnets or even multiple VCNs.
    * This implies that the seclists and route tables on one VCN might not stop traffic to an instance which has an alternate network path through a second vNIC attachment
  * Each vNIC has a primary private IP address, as well as upto 31 secondary private IP addresses. **total=32**
  * The primary private IP address is assigned at boot time and the secondary IP addresses are only assigned after launch.
  * A secondary private IP can be moved from one vNIC to antother
    * Helpful in failover. If one instance fails, the IP can be moved to another identical instance.

#### Public IP

Public IPs are reachable from the internet and can be assigned to elements in a public subnet.

* Public IPs are attaced to a Private IP - meaning that for an element to use a Public IP it should already have a Private IP.
* Resources can be assigned multiple Public IPs corresponding to Private IPs and also across vVNICs attached to the element
* Public IPs are assigned to
  * Instances
  * NAT gateway - You cannot choose or edit the IP allocated
  * Internet Gateway - Cannot be changed or edited.
  * DRG - Cannot edit the IP address allocated.
  * ATP & AWD - Same - there are Oracle provided and managed, customer cannot edit or manage these IPs.
  * OKE - Same - The IP addresses are  managed by Oracle and cannot be edited by the user.
* Public IPs are __Ephemeral__ or __Reserved__
* No charges for using a Public IP addresses even they are not attached to any isntance.

### Internet Gateway

* They provide a mechanism for public internet traffic to enter/exit a VCN.
* Without an Internet Gateway, a VCN is isolated from the internet even if it has a public IP.
* Is attached to a VCN, meaning **its a regional construct**
* There can be **only one IG per VCN**.
* A route needs to be defined in the route table for outbound traffic to target the IG.

### Route Table

A route table is a set of routing instuctions for network traffic in a VCN.

* Each **subnet has a single route table**
  * A single route table can be shared among many subnets and across ADs in the same region
* Being attahed to a subnet makes the route table an **AD scoped** element.
* A route rule is used when the destination address is outside the VCN's CIDR.
  * You can explicitly provide a Private IP as the target for a route rule for cases when you want to say direct internet traffic to a specific internal destination (like a firewall) that will then forward the request outward
* No route rules are requrired when routing traffic within the VCN itself.
* Route rules are required when using gateways or connections

### NAT Gateway

Its used for connecting a private network/subnet to the internet.

* The hosts in the subnet do not need to assigned a public IP address.
* A Route must exist that maps a  source CIDR to the the NAT-G for traffic.
* NAT-G gets a public IP, and this IP cannot be changed, and customer's reserved public IP cannot be used
* Traffic can go out of the subnet to the internet and return, 
  * However nothing from the internet can connect to any thing on the private subnet.
* There can be multiple NAT gateways per VCN (unlike a single Internet Gateway per VCN) 
  * However a given subnet in the VCN can only route traffic to a single NAT gateway.
  * Having multiple NAT gateways makes it possible for the internet destination to distinguish between subnets based on the public IP of the NAT gatyeway.
* When switching from an IG to a NAT-G, the original public subnets cannot be made private. The public IPs in the subnet can be deleted though.
* A NAT-G can be used only by entities within the VCN
  * If the VCN is peered, or connected to CPE with IPsec/FastConnnect, those sysyems cannot use the NAT-G

### Service Gateway

Its a special gateway to route traffic from a subnet to **public OCI services**.

* Service gateways are regional, they provide access to Oracle services in the same region as the VCN
* There is one service gateway per VCN.
  * You can control which subnets in the VCN can use the SG
  * Various subnets in the VCN can choose to use the SG or not, based on the route rules for the subnet.
* With a service gateway, there in no need for an IG or NAT. 
* Instances in private subnets can call *public* OCI services, over the interal network fabric
* Although the service is public, since its on OCI itself, a Service Gateway routes the traffic internally.
* Traffic over the service gateway does not traverse the internet.
* Object storage is the only service supported now.
* RouteTables and SecurityLists needs to be updated, but instead of a destination CIDR block, you specify the  destination service.
* Instances in Peered VCNs or CPE (over IPSecVPN/FastConnect) cannot use the service gateway.
  * Sevices like Object storage support **Public Peering** over FastConnect/IPSecVPN for CPEs to talk to public services over a private network

#### CIDR Labels

CIDR Labels is an easy way to target oracle services in route rules, without knowing the actual IP addresses.
This also makes sure that the routes work even if the CIDRs change in the future.

* OCI <*region*> Object Storage - only access to object storage in the region
* All <*region*> Services in Oracle Services Network - much wider access, including object storage.

* A Service Gateway can use only one CIDR label. 
* Likewise, a RouteTable can have only one RouteRule with a CIDR label. 
  * You cannot have two rules targeting the two CIDR labels. 
* The route rule that targets a **Service Gateway** needs to specify the same CIDR label enabled in the Service Gateway.
  * The console ensure this rule. 

### Dynamic Routing Gateway

A DRG is used to connect your VCN to your OnPrem network or to a remote VCN.

* Its a standalone object, but has a **1:1 relationship with a VCN**. A VCN can be attached to only one DRG at a time and vice versa.
* The DRG needs to reside in a compartment for access control reasons.
* 3 Reasons to have a DRG
  * CPE connectivity over IPSecVPN
  * CPE connectivity over FastConnect (Private peering)
  * Cross Region connectivity between to OCI regions (Remote Peering).
* It can be atached to an VCN and detached from it any time
  * Attaching the DRG to an VCN creates an DRGAttachment object, and deleting this object detaches it.
* Connects traffic from a private subnet to destinations that are in the a remote network, that are not internet.
* The traffic is entirely private, but its crossing the boundary of the VCN
* Connectivity is provided by either and IPSec VPN or FastConnect(dedicated channel)
* Using a DRG requires the subnet to have an entry in the route table.  


### VCN Peering

* Connects two VCNs with each other over the Oracle private network fabric
* Avoid traffic being routed over internet and does nto require IG or NAT-G
* Peered networks should not have overlapping subnets or addresses
* Peering can be done between two separate tenancies as well.

### Local Peering Gateway

This is a mechanism to connect two separate VCNs in the same region.
You may have different VCNs in the same region and if you use an LPG to establish connectivity between.
Remember that VCNs are regional in scope, so "Local" in this context means that its local within the same region.

* LPG is defined for a specific VCN.
* To connect two different VCNs,
  * Each VCN must have an LPG.
  * To setup LPGs and create connection, specific IAM policies are needed.
  * On each VCN, a route rule is defined such that 
    * The route destination is the VCN's own LPG
    * The destination CIDR is the CIDR of the other VCN.
    * It effectively reads that for addresses in the destination CIDR, use this LPG.
  * The LPGs are then connected to each other.
  * For a VCN to talk to 2 other VCNs in the same region, different LPG pairs are used.
  * VCNs in a peering relationship cannot have overlapping CIDRs
    * The route table rules do not apply to desitnations within the VCN.
    * Overlapping addresses will not resolve to the remote destinations

### Remote Peering 

This is similar to local VCN peering, however applied to VCNs in different regions.
The main purpose for this is to utilize the private communication fabric offered by the
Oracle backbone network, instead of sending traffic over the internet.

* Unlike the LPG case, there is an additional entity at play here - the Remote Peering Connection

* Connects two VCNs in OCI that are in different regions.
* Traffic is private and goes over the Oracle channel between the regions
* There is no _Remote Peering Gateway_, a pair of DRGs are used.
* The DRGs are connected to each other using a **Remote Peering Connection**
  * There is only one DRG for a VCN at a time
  * To connect to multiple remote networks, the DRG uses separate **RPC**s. 
  * This is unlike LPGs where the VCN can have multiple LPGs, so for multiple networks, different LPG pairs can be used within the same VCN.
* The traffic flow is 
  * Source vNIC-1 -> DRG-1 -> DRG-2-> Target vNIC-2
* Specific IAM policies are required. There are requestor as well as acceptor policies
* Common for cross-region replication scenarios

> **Confusing Terms**
>    * **Private Peering**
>      * Customer extending their data-center into OCI.
>      * Connection established through IPSecVPN or FastConnect
>    * **Public Peering**
>      * Customer is making calls to OCI public services like Object Storage, over a dedicated private channel
>      * Connection established through FastConnect
>      * Does not require a DRG in OCI, since the connectivity is established with an OCI Managed public service
>    * **Remote Peering**
>      * Connectivity between VCNs in two OCI regions
>      * Unrelated to scenarios involving customer's on-prem resources. 
>    * **Local Peering**
>      * Connectivity between two VCNs in the same region.
>      * Unrelated to scenarios involving customer's on-prem resources.

### Transit Routing

This is a case where the CPE needs to connect to multiple VCNs in the same region.

* Uses a single FastConnect or IPSec VPN to connect your on-premises network with multiple VCNs in the same region, in a hub-and-spoke layout
* The VCNs must be in the same region but can be in different tenancies. 
* The CIDR blocks of the various subnets of interest in the on-premises network and VCNs must not overlap.
* The _hub_ uses a DRG to communicate with the CPE, and LPGs to communicate with the _spoke_ VCNs
* The _hub_ direct traffic using a route table on the **DRG attachment** (usually route tables are on the subnets)
* Note that a route table associated with a DRG attachment can contain only rules that use a local peering gateway (LPG) on the attached VCN as the target.

### Security Lists

* These are objects that are attached to subnets. 
  * When attached, they control the traffic that can flow in and out of all elements in the subnet.
  * A subnet can have a max of upto 5 seclits
* Seclists are applied whether an instance is communicating with another host on the same VCN or not.
* Rules can be stateful or stateless. Default is __stateful__
* Stateful rules use connection tracking.
  * this means that if there is an ingress rule (allow any connection to :80)
  * then an egress rule is not required. OCI tracks the source IP and port and alows the response to go out. 
* Stateless rules do not use connection tracking. The fact that the rule is stateless means that connection tracking is disabled.
  * Stateless rules require that a corresponding egress rule be creatd or response traffic will be blocked.
  * Better performance than stateful rules.

### Default Components for the VCN

VCN comes with some default objects that cannot be deleted, but can be edited.

* Default Route Table
* Default Security List
* Default DHCP options
* The default components can be edited but cannot be deleted.

### Internal DNS

There is an internal DNS within OCI that resolves the FQDNs and other domain names. You can customize this using the __DHCP Options__ object

* DNS Labels can be set for the various network elements forming the FQDN for an instance
  * <hostname>.<subnet_dnslabel>.<vcn_dnslabel>.oraclevcn.com
  * These FQDNs __always__ resolve to the private IP address even if the instnace has a Public IP address

## Edge Security

## Questions

* Remote Peering along with On Prem Connectivity
  * Since a VCN can be connected to a single DRG at a time and a DRG can be conneced to a single VCN at a time, How do you connect a VCN to an OnPrem network as well as a different Region ?
    * Potential answer : use a DRG for either one (say, Remote peering). 
    * Then create a separate VCN in the same region  and use that to do the other (Onprem connection)
    * Now do local peering between the VCNs in the same region.
    * The VCN might need to have an overlapping subnet with the Customer Premises Equipment (CPE) and a route rule that sends every thing to the CPE.  