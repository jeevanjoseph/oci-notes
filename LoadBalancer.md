# Load Balancing

- [Load Balancing](#load-balancing)
  - [Overview](#overview)
  - [Key diiferentiators](#key-diiferentiators)
  - [Public Loadbalancer service](#public-loadbalancer-service)
  - [Private Loadbalancer serivce](#private-loadbalancer-serivce)
## Overview

OCI provides load balancing services, that can be used for service discovery, fault tolerance and HA, and L7 routing.
OCI load balancers can do health checks on "backend services" and use a variety of algorithms to route traffic between the healthy nodes.

* Support protocols like
  * TCP, HTTP, HTTP/2, Websocket. each protocol requires a separate listener. 
  * SSL features like SSL termination, SSL Tunnneling
  * Content based routing and Session Persistence (sticky sessions)

## Key diiferentiators

  * You get an actual dedicated public IP, not just a CNAME to a shared LB.
  * Provisioned bandwidth - 100Mbps/400Mbps/8Gbps
    * this means that the capacity is always avaialble - no need to "warm" your LB (like providers that give you bandwidth based on traffic)
    * Single LB for Layer 4 (TCP) and Layer 7(HTTP)  traffic.

## Public Loadbalancer service

  * Used as an entry point for external (internet) traffic. Has a public IP address.
  * Actually a set of 2 Loadbalancers behind a floating VIP.
  * Its regional in scope.
  * requires at least 2 public subnets in two separate ADs or a single regional subnet.
    * Provides redundancy, and if one fails the other takes over - the public IP is a floating VIP.
    * The LB service takes 2 IPs from the subnet(s) - one for each LB instance.
    * User cannot designate a "primary" or secondary.
    * Deployed in Active/standby fashion - cannot be changed.
    * The secondary is transparent - the service is exposed as a single load balancer.
    * In single AD regions, only one subnet is required. There is no failover in case of AD failure.

## Private Loadbalancer serivce

  * These are AD local, and uses a single subnet but are sill deployed in pairs to protect aganist a rack failing (different fault domains)
  * Can be regional or AD specific depending on the subnet used.
  * Its accessible only from within the VCN where the subnet is, and further restricted by security rules.
  * It takes 3 IPs from the subnet
    * One private IP that floats between the two
    * One private dedicated IP for each load balancer instance.
    * In case of an AD outage:
      * if the LB is created in a regional subnet :
        * where the region has multiple ADs, failover to another AD is available
        * whre the region has a single AD, no failover.
      * If the LB is created in an AD specific subnet, then there is no failover

##Creation & Management

  * All LBs have a 
    * __Backend Set__ - set of servers to which traffic is to be forwarded
      * The servers can be in multiple ADs or anywhere as long as there is a network path and the subnet security lists allow traffic to come in.
    * __Health Check__ policy
      * Both TCP and HTTP health checks are available.
      * Health can be __OK, Warning, Critical, Unknown__
      * Health checks are done at 3 min interval. Cannot be finer.
    * __Load Balancing__ policy
      * Round Robin
      * IP Hash - uses the source IP address as a hash key
      * Least Connections 
      * Policies are applicable only to TCP and __non-sticky__ HTTP traffic. If __session persistence is enabled__, then cookie info is used for routing.
    * SSL config (opt)
    * Session persistence config (opt)

##Limitations

  * Cannot change shape dynamically - cannot make a 100Mbps LB in to a 400Mbps LB
  * Cannot convert an AD specific LB (created with AD specific Subnets) to a regional LB
  * Each LB has:
    * One IP address
    * 512 backend servers per backend set
    * 1024 backend servers total
    * 16 listeners (each LB ip can listen on 16 ports)
      * Each listener port points to one backend set =  * 16 backend sets.
