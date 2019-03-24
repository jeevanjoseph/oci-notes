## Virtual Cloud Networks

My notes on network elements in OCI.

### CIDR notation

Recollect the basics :
-  An IP address is 32 bits. 
    - The 32 bits are divided in to four 8-bit octets.
    - An Ip address in binary looks like this :
        - 00000000.00000000.00000000.00000000 to 
        - 11111111.11111111.11111111.11111111
    - Since thats obviously inconvinient, we represent each of those 8-bit segments (octets) in decimal.
    - 00000000 => 0 and 11111111 => 255 in decimal form. 
    - So in decimal form we can represent the same range as 0.0.0.0 - 255.255.255.255
CIDR Notation
    - CIDR represents chunk of contigous IP addresses => CIDR Block
    - The notation simply denotes an IP addres with the number of bits that are fixed. 
    - The remaining number of bits gives the possibilities for IP addresses in that block = the size of the CIDR block
        - 10.0.0.0/30 => Remember that there are only 32 bits in an IPv4 address. 
            - This CIDR says 30 bits are fixed, so only 2 bits can change => we have 2^2 = 4 possible addresses
            - They range from 10.0.0.0 - 10.0.0.3
            - So a CIDR Block 10.0.0.4/30 can represent another non-overlapping CIDR Block - 10.0.0.4-10.0.0.7 (4 IPs)
    - So a 10.0.0.0/24 CIDR block has 24 bits reserved, and we have 32-24= 8 bits available to us for allocation
        - This gives us 2^8 = 256 addresses, from  10.0.0.0 - 10.0.0.255
    - Another example 10.0.0.0/20
        - 20 bits are blocked, leaving 12 bits for the addresses, or 2^12 = 4096 addresses.
        - Of the 20 bits blocked => first two octets, and the first 4 bits of the 3rd octet, leaving 4 bits in the 3rd octect
            - **00010001.00000000.0000**_0000.00000000_
        - The third octect can range from 0000 - 1111 => 0-15
        - So the 4096 addresses will be 10.0.0.255, 10.0.1.255, .... 10.0.15.255

### VCN

VCNs represent distinct networks, in the tenancy with their own CIDR blocks.
- They are regional. Two regions, require two VCNs, one per region (minimum)
- CIDR ranges can be /16 biggest network, to /30 smallest network. (10.0.0.0/8, 172.16/12 are in RFC, but not supprted by VCN)
- VCN reserves the first **TWO** IP addresses and the last IP address in each CIDR block. 
    - Normally the first address .0 is used by the network itself, and the last one .255 is used for broadcast.
    - OCI in addition to this reserves one more, the second IP in the block.
    - So a /30 block will yeild **one** usable address, instead of 4. 

### Subnets

Subnets in an VCN is the logical equivalent of an actual subnet. 
They divide the IP namespace of the VCN in to distinct groups that can be managed indepdently. eg: a DB subnet can be isolated from a web-tier subnet.
- Are an AD level construct, and can be **Public** or **Private**
- **Public** subnets allow the use of public IP addresses on vNICs inside the subnet.
- As subnets are AD specific, the VCN CIDR block is divided in to multiple subnets at the AD level.
    - The subnet address range should be within the VCN block
    - Subnet addressing ranges cannot overlap
    - 
### Internet Gateway

They provide a mechanism for public internet traffic to enter/exit a VCN. 
Without an Internet Gateway, a VCN is isolated from the internet even if it has a public IP.
- Is attached to a VCN, meaning its a regional construct
- There can be only one IG per VCN.
- A route needs to be defined in the route table for outbound traffic to target the IG.

### Route Table

A route table is a set of routing instuctions for network traffic in a VCN.
- Each subnet has a single route table
- Being attahed to a subnet makes the route table an AD scoped element.
- A route rule is used when the destination address is outside the VCN's CIDR.
- No route rules are requrired when routing traffic within the VCN itself.
- Route rules are required when using gateways or connections

### NAT Gateway

Its used for connecting a private network/subnet to the internet.
- The hosts in the subnet do not need to assigned a public IP address.
- Traffic can go out of the subnet to the internet and return, 
    - However nothing from the internet can connect to any thing on the private subnet.
- There can be multiple NAT gateways per VCN (unlike a single Internet Gateway per VCN) 
    - However a given subnet in the VCN can only route traffic to a single NAT gateway.

### Service Gateway

Its a special gateway to route traffic from a subnet to **public OCI services**.
- There is one service gateway per VCN.
- Although the service is public, since its on OCI itself, a Service Gateway routes the traffic internally.
- Traffic over the service gateway does not traverse the internet.
- Object storage is the only service supported now.
- Route table needs to be updated, but instead of a destination CIDR block, you specify the  destination service.

### Dynamic Routing Gateway

A DRG is used to connect your VCN to your OnPrem network. 
- Its a standalone ibject, but has a 1:1 relationship with a VCN
- Connects traffic from a private subnet to destinations that are in the a remote network, that are not internet.
- The traffic is entirely private, but its crossing the boundary of the VCN
- Connectivity is provided by either and IPSec VPN or FastConnect(dedicated channel)
- Using a DRG requires the subnet to have an entry in the route table.  

### Local Peering Gateway.

This is a mechanism to connect two separate VCNs in the same region. 
You may have different VCNs in the same region and if you use an LPG to establish connectivity between.
Remember that VCNs are regional in scope, so "Local" in this context means that its local within the same region.

- LPG is defined for a specific VCN.
- To connect two different VCNs, 
    - Each VCN must have an LPG.
    - On each VCN, a route rule is defined such that 
        - The route destination is the VCN's own LPG
        - The destination CIDR is the CIDR of the other VCN.
        - It effectively reads that for addresses in the destination CIDR, use this LPG.
    - The LPGs are then connected to each other.
    - For a VCN to talk to 2 other VCNs in the same region, different LPG pairs are used.
    - VCNs in a peering relationship cannot have overlapping CIDRs
        - The route table rules do not apply to desitnations within the VCN.
        - Overlapping addresses will not resolve to the remote destinations

### Remote Peering 

This is similar to local VCN peering, however applied to VCNs in different regions.
The main purpose for this is to utilize the private communication fabric offered by the 
Oracle backbone network, instead of sending traffic over the internet. 

Unlike the LPG case, there is an additional entity at play here - the Remote Peering Connection

- Connects two VCNs in OCI that are in different regions.
- Traffic is private and goes over the Oracle channel between the regions
- There is no _Remote Peering Gateway_, a pair of DRGs are used.
- The DRGs are connected to each other using a **Remote Peering Connection**
-


## Questions
- Remote Peering along with On Prem Connectivity
    - Since a VCN can be connected to a single DRG at a time and a DRG can be conneced to a single VCN at a time, How do you connect a VCN to an OnPrem network as well as a different Region ?
        - Potential answer : use a DRG for either one (say, Remote peering). 
        - Then create a separate VCN in the same region  and use that to do the other (Onprem connection)
        - Now do local peering between the VCNs in the same region.
        - The VCN might need to have an overlapping subnet with the Customer Premises Equipment (CPE) and a route rule that sends every thing to the CPE.  