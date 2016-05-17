---
layout:   post
title:    'Providing globally routed IPv6 adresses to your OpenStack tenants^Wprojects'
date:     2016-05-17 11:20:00 +0200
author:   Dr. Jens Rosenboom
categories: openstack neutron networking
---

# Motivation

With the runout of available IPv4 space, the importance of being able to
use IPv6 addresses instead of or at least in addition to IPv4 addresses has been ever increasing.
Obviously this also applies to virtualized hosts deployed in OpenStack clouds.
Now when it comes to the actual implementation within Neutron, there is a
significant difference in the treatment of IPv4 and IPv6 that we will have to
take into account for our deployment.

# Legacy

OpenStack was born late enough in the history of the internet such that the
runout of IPv4 was clearly visible on the horizon right from the start.
Therefore, the way "private" per-project networks were built was based on the
assumption that projects could choose any IPv4 range for their network that they
happened to like, because it would have no direct connectivity to the public
internet anyway. 

Then, in order to still allow access to instances from the internet, the concept
of *Floating IP addresses* was introduced, allowing projects to receive globally
routed IPv4 addresses one at a time and setup NAT rules on a router automatically
such that traffic towards this address would end up on their instance within its
private network.

# A new hope

With IPv6 the use of NAT is strongly deprecated with the intention of allowing
direct end-to-end connectivity between hosts. Thus the Neutron implementation
does not provide a mechanism to create floating IPv6 addresses.

As a consequence, projects will want globally routed IPv6 adresses directly connected
to their instances. As this will be difficult to achieve if you continue to allow
your projects to choose their own favorite range of addresses, there are two
possible solutions:

1. Set up a shared network to which your instances will be directly connected and
      configure this network with a public IPv6 prefix (*provider network*). This
      way instances will get a public IPv6 address that they can use without any
      restriction.
2. Configure a *subnet-pool* containing a range of public IPv6 prefixes, so that
      projects may configure their own networks by requesting a slice from that
      subnet-pool instead of choosing their own.

Option 1 has some drawbacks, though:

- It places all projects into a single shared network, complicating things like
     per-project firewall rules or rDNS management.
- If you want to dual-stack your instances (i.e. provide them with IPv4 and IPv6
     connectivity at the same time), you either need to use two different networks
     and attach your instances to both of them, or you will need to also assign a
     public IPv4 address to every instance, which maybe considered a rather wasteful
     approach to dealing with this scarce resource.

# New obstacles

So we end up with wanting to provide connectivity to project networks that are
using prefixed allocated from a subnet-pool we previously defined. Again there
are multiple solutions available:

0. Manually create routes from your external gateways towards the project routers
      once the project networks have been set up. Did I really just list this as an option?
      Obviously this does not scale to anything but maybe a small POC lab and even
      there inserting some manual intervention into a process that should be able
      to be orchestrated in a dynamic fashion should best be avoided.
1. Use *prefix delegation*. This is a way of handing out slices out of a predefined
      subnet-pool where the delegation process is not done internally by Newton, but
      instead we would set up an external DHCPv6 server providing this functionality.
      That server will then also setup routing between itself and the project routers.
      Again, there are some drawbacks:

      - There doesn't seem to be an easy way to deploy the DHCPv6 server in a redundant
        fashion. The only feasible approach would probably be to use some mirrored
        setup involving DRBD/pacemaker/corosync stuff.
      - Due to the dynamic nature of prefix delegation, Neutron doesn't allow to use
        to dhcpv6-stateful method of handing out prefixes to instances.
2. Use *dynamic routing*. Here we deploy some additional agents that will setup
      BGP sessions towards our external routers, announcing the prefixes for the
      project network and thus dynamically establishing the connectivity we want.
      There still are some issues here:
      
      - Currently only 16bit ASNs are supported, which is another scarce resource
        nearing exhaustion. So if you happen to be using some ASN > 65535 for the
        remainder of your network, you will have to insert some fiddling with
        private ASNs, which is almost, but not quite, as nasty as having to do NAT.
      - The dynamic routing agent will want to setup the BGP session from the top
        level routing space on your network nodes, so you will need to have some
        kind of connectivity between these nodes and your external routers. This
        is something that one would usually want to avoid, a better solution would
        be to deploy a dedicated network namespace here, similar to what the L3 agent
        does.

Despite these issues, solution 2 seems to be best suited for deploying a public
OpenStack cloud service.

# Welcome to the real world

So let us take a look at how to configure your OpenStack cluster to utilize dynamic
routing. We start by setting up an address-scope that will be used by the BGP agent in order
to select the set of prefixes to be announced within the BGP sessions:

    # neutron address-scope-create --shared address-scope-ip6 6
    Created a new address_scope:
    +------------+--------------------------------------+
    | Field      | Value                                |
    +------------+--------------------------------------+
    | id         | 040256da-f8a9-4009-9062-043b70d9e8a6 |
    | ip_version | 6                                    |
    | name       | address-scope-ip6                    |
    | shared     | True                                 |
    | tenant_id  | 4a0a30294e954953a41a9791d4e7f437     |
    +------------+--------------------------------------+

Next we create the subnetpool that our projects will use to configure their subnets
with:

    # neutron subnetpool-create --address-scope address-scope-ip6 --shared --pool-prefix 2001:db8:1234::/48 --default-prefixlen 64 --max-prefixlen 64 --is-default true default-pool-ip6                                                                                                        
    Created a new subnetpool:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | address_scope_id  | 040256da-f8a9-4009-9062-043b70d9e8a6 |
    | created_at        | 2016-04-27T08:19:27                  |
    | default_prefixlen | 64                                   |
    | default_quota     |                                      |
    | description       |                                      |
    | id                | 84367d47-4b17-4ffc-9240-e01695e604eb |
    | ip_version        | 6                                    |
    | is_default        | True                                 |
    | max_prefixlen     | 64                                   |
    | min_prefixlen     | 64                                   |
    | name              | default-pool-ip6                     |
    | prefixes          | 2001:db8:1234::/48                   |
    | shared            | True                                 |
    | tenant_id         | 4a0a30294e954953a41a9791d4e7f437     |
    | updated_at        | 2016-04-27T08:19:27                  |
    +-------------------+--------------------------------------+

Our public network, i.e. the network that the gateway ports of the project routers
will be connected to, needs to be configured with the same address_scope. As the
address scope can only be assigned via subnetpools and not directly, we either have
to assign the IPv6 subnet on the public network from the shared subnetpool we just
created, or we create a second subnetpool containing the specific prefix that we
want to use for our public network:

    # neutron subnetpool-create --address-scope address-scope-ip6 --pool-prefix 2001:db8:4321:42::/64 --default-prefixlen 64 public-pool
    Created a new subnetpool:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | address_scope_id  | 040256da-f8a9-4009-9062-043b70d9e8a6 |
    | created_at        | 2016-04-27T08:22:19                  |
    | default_prefixlen | 64                                   |
    | default_quota     |                                      |
    | description       |                                      |
    | id                | 4240ea15-ee8e-4af4-ad68-dc1c97e2b54d |
    | ip_version        | 6                                    |
    | is_default        | False                                |
    | max_prefixlen     | 128                                  |
    | min_prefixlen     | 64                                   |
    | name              | public-pool                          |
    | prefixes          | 2001:db8:4321:42::/64                |
    | shared            | False                                |
    | tenant_id         | 4a0a30294e954953a41a9791d4e7f437     |
    | updated_at        | 2016-04-27T08:22:19                  |
    +-------------------+--------------------------------------+

Now we can create our public network and the IPv6 subnet on it:

    # neutron net-create --provider:network_type flat --provider:physical_network external --router:external=True public
    <output skipped>
    # neutron subnet-create --name public-ip6 --ip_version 6 --subnetpool public-pool public
    Created a new subnet:
    +-------------------+---------------------------------------------------------------------------------+
    | Field             | Value                                                                           |
    +-------------------+---------------------------------------------------------------------------------+
    | allocation_pools  | {"start": "2001:db8:4321:42::2", "end": "2001:db8:4321:42:ffff:ffff:ffff:ffff"} |
    | cidr              | 2001:db8:4321:42::/64                                                           |
    | created_at        | 2016-04-27T08:23:33                                                             |
    | description       |                                                                                 |
    | dns_nameservers   |                                                                                 |
    | enable_dhcp       | True                                                                            |
    | gateway_ip        | 2001:db8:4321:42::1                                                             |
    | host_routes       |                                                                                 |
    | id                | 98cd2ae1-cf7d-48f3-b5e0-c94efe3c2562                                            |
    | ip_version        | 6                                                                               |
    | ipv6_address_mode |                                                                                 |
    | ipv6_ra_mode      |                                                                                 |
    | name              | public-ip6                                                                      |
    | network_id        | 75d2c0d9-4343-4ac7-8dbe-b72893267201                                            |
    | subnetpool_id     | 4240ea15-ee8e-4af4-ad68-dc1c97e2b54d                                            |
    | tenant_id         | 4a0a30294e954953a41a9791d4e7f437                                                |
    | updated_at        | 2016-04-27T08:23:33                                                             |
    +-------------------+---------------------------------------------------------------------------------+

Verify that the address_scope for IPv6 indeed got set for this network via creating the subnet from the
proper subnetpool:

    # neutron net-show public
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        | nova                                 |
    | created_at                | 2016-04-26T09:44:10                  |
    | description               |                                      |
    | id                        | 75d2c0d9-4343-4ac7-8dbe-b72893267201 |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        | 040256da-f8a9-4009-9062-043b70d9e8a6 |
    | is_default                | False                                |
    | mtu                       | 1500                                 |
    | name                      | public                               |
    | provider:network_type     | flat                                 |
    | provider:physical_network | external                             |
    | provider:segmentation_id  |                                      |
    | router:external           | True                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   | 98cd2ae1-cf7d-48f3-b5e0-c94efe3c2562 |
    | tags                      |                                      |
    | tenant_id                 | 4a0a30294e954953a41a9791d4e7f437     |
    | updated_at                | 2016-04-26T09:44:10                  |
    +---------------------------+--------------------------------------+

# View from the other side

We have now prepared everything on the admin side of things, so let our user start to configure their networking:

    $ neutron net-create pronet
    Created a new network:
    +-------------------------+--------------------------------------+
    | Field                   | Value                                |
    +-------------------------+--------------------------------------+
    | admin_state_up          | True                                 |
    | availability_zone_hints |                                      |
    | availability_zones      |                                      |
    | created_at              | 2016-04-27T12:45:53                  |
    | description             |                                      |
    | id                      | ab20362b-b4ca-4490-9793-8b5759a34ee3 |
    | ipv4_address_scope      |                                      |
    | ipv6_address_scope      |                                      |
    | mtu                     | 1450                                 |
    | name                    | pronet                               |
    | router:external         | False                                |
    | shared                  | False                                |
    | status                  | ACTIVE                               |
    | subnets                 |                                      |
    | tags                    |                                      |
    | tenant_id               | a1380d88a6f44898be538f7b2ba1814f     |
    | updated_at              | 2016-04-27T12:45:53                  |
    +-------------------------+--------------------------------------+

In order to add some IPv6 prefix, simply request one from the default pool:

    $ neutron subnet-create --name subnet6 --ip_version 6 --use-default-subnetpool --ipv6-address-mode slaac --ipv6-ra-mode slaac pronet
    Created a new subnet:
    +-------------------+---------------------------------------------------------------------------------+
    | Field             | Value                                                                           |
    +-------------------+---------------------------------------------------------------------------------+
    | allocation_pools  | {"start": "2001:db8:1234:1::2", "end": "2001:db8:1234:1:ffff:ffff:ffff:ffff"}   |
    | cidr              | 2001:db8:1234:1::/64                                                            |
    | created_at        | 2016-04-27T12:50:27                                                             |
    | description       |                                                                                 |
    | dns_nameservers   |                                                                                 |
    | enable_dhcp       | True                                                                            |
    | gateway_ip        | 2001:db8:1234:1::1                                                              |
    | host_routes       |                                                                                 |
    | id                | 796d95f8-20e9-40b2-82df-1d50422a5c15                                            |
    | ip_version        | 6                                                                               |
    | ipv6_address_mode | slaac                                                                           |
    | ipv6_ra_mode      | slaac                                                                           |
    | name              | subnet6                                                                         |
    | network_id        | ab20362b-b4ca-4490-9793-8b5759a34ee3                                            |
    | subnetpool_id     | 84367d47-4b17-4ffc-9240-e01695e604eb                                            |
    | tenant_id         | a1380d88a6f44898be538f7b2ba1814f                                                |
    | updated_at        | 2016-04-27T12:50:27                                                             |
    +-------------------+---------------------------------------------------------------------------------+

We want outside connectivity, so we create a router, add an interface into our project net and set the gateway to be
the public network:

    $ neutron router-create router1
    Created a new router:
    +-------------------------+--------------------------------------+
    | Field                   | Value                                |
    +-------------------------+--------------------------------------+
    | admin_state_up          | True                                 |
    | availability_zone_hints |                                      |
    | availability_zones      |                                      |
    | description             |                                      |
    | external_gateway_info   |                                      |
    | id                      | 00991f6c-882b-409a-952c-bb358098555c |
    | name                    | router1                              |
    | routes                  |                                      |
    | status                  | ACTIVE                               |
    | tenant_id               | a1380d88a6f44898be538f7b2ba1814f     |
    +-------------------------+--------------------------------------+
    $ neutron router-interface-add router1 796d95f8-20e9-40b2-82df-1d50422a5c15
    Added interface 703a3bdb-3174-46d4-bc70-a378d2ad41b7 to router router1.
    $ neutron router-gateway-set router1 public
    Set gateway for router router1

Finally we boot an instance and verify that it gets an IPv6 address assigned:

    $ nova boot --flavor 1 --image cirros vm1
    $ nova list
    +--------------------------------------+------+--------+------------+-------------+---------------------------------------------+
    | ID                                   | Name | Status | Task State | Power State | Networks                                    |
    +--------------------------------------+------+--------+------------+-------------+---------------------------------------------+
    | 17b2ac04-9a17-45ff-be30-401aa8331a66 | vm1  | ACTIVE | -          | Running     | pronet=2001:db8:1234:1:f816:3eff:fe53:f89e  |
    +--------------------------------------+------+--------+------------+-------------+---------------------------------------------+
    $ nova console-log vm1 | grep -A1 -B1 2001
    eth0      Link encap:Ethernet  HWaddr FA:16:3E:53:F8:9E  
              inet6 addr: 2001:db8:1234:1:f816:3eff:fe53:f89e/64 Scope:Global
              inet6 addr: fe80::f816:3eff:fe53:f89e/64 Scope:Link

Note that most cloud images are built to insist on receiving an IPv4 address via DHCP
while booting, so in order to avoid the resulting delay, you could add an IPv4 subnet
to your project net.

# Getting into advertising

Up to this point most of the things we have done have been pretty standard, so let's
get to the interesting stuff now. First we have to update our ``neutron.conf`` a bit:

    [DEFAULT]
    # You may have other plugins enabled here depending on your environment
    # Important thing is you add "bgp" to the list
    service_plugins = bgp, router
    # In case you run into issues, this will also be helpful
    debug = true

You need to restart your ``neutron-server`` process(es) in order activate the plugin.
Now we can create our first BGP speaker. We set the IP version to 6, select some
private ASN that we can use for our POC and disable advertising floating IPs, as we
do not have these for IPv6 anyway:

    # neutron bgp-speaker-create --ip-version 6 --local-as 65001 --advertise-floating-ip-host-routes false bgp1
    Created a new bgp_speaker:
    +-----------------------------------+--------------------------------------+
    | Field                             | Value                                |
    +-----------------------------------+--------------------------------------+
    | advertise_floating_ip_host_routes | False                                |
    | advertise_tenant_networks         | True                                 |
    | id                                | 09fc7063-89e1-4949-812d-b52eb7eee430 |
    | ip_version                        | 6                                    |
    | local_as                          | 65001                                |
    | name                              | bgp1                                 |
    | networks                          |                                      |
    | peers                             |                                      |
    | tenant_id                         | 4a0a30294e954953a41a9791d4e7f437     |
    +-----------------------------------+--------------------------------------+

We add our public network to this speaker, indicating that we want to advertise all those
tenant networks here, which have a router with the public network as gateway. We also
verify that our tenant network gets listed for advertisement:

    # neutron bgp-speaker-network-add bgp1 public
    Added network public to BGP speaker bgp1.
    # neutron bgp-speaker-advertiseroute-list bgp1
    +----------------------+---------------------+
    | destination          | next_hop            |
    +----------------------+---------------------+
    | 2001:db8:1234:1::/64 | 2001:db8:4321:42::5 |
    +----------------------+---------------------+

Assume our external router has the address 2001:db8:4321:e0::1, we configure it as
BGP peer and add it to our BGP speaker:

    # neutron bgp-peer-create --peer-ip 2001:db8:4321:e0::1 --remote-as 65001 bgp-peer1
    Created a new bgp_peer:
    +-----------+--------------------------------------+
    | Field     | Value                                |
    +-----------+--------------------------------------+
    | auth_type | none                                 |
    | id        | f6461ba4-aac9-41a9-8783-b93570e0a768 |
    | name      | bgp-peer1                            |
    | peer_ip   | 2001:db8:4321:e0::1                  |
    | remote_as | 65001                                |
    | tenant_id | 4a0a30294e954953a41a9791d4e7f437     |
    +-----------+--------------------------------------+
    # neutron bgp-speaker-peer-add bgp1 bgp-peer1
    Added BGP peer bgp-peer1 to BGP speaker bgp1.

# The missing link

So all that it left now is to setup the agent that will start the BGP session and
advertise the prefixes to the outside world. On the network node, install the
necessary packages, e.g. for Ubuntu Xenial:

    # apt install neutron-bgp-dragent python-ryu

This will also add a configuration file at ``/etc/neutron/bgp_dragent.ini`` which
needs some tuning:

    [BGP]
    # BGP speaker driver class to be instantiated. (string value)
    bgp_speaker_driver = neutron.services.bgp.driver.ryu.driver.RyuBgpDriver

    # 32-bit BGP identifier, typically an IPv4 address owned by the system running
    # the BGP DrAgent. (string value)
    bgp_router_id = 10.11.12.13

Again you will have to restart the agent in order for it to pick up the new
configuration. If all goes well, the agent should register itself in your setup:

    # neutron agent-list
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+
    | id                                   | agent_type                | host         | availability_zone | alive | admin_state_up | binary                    |
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+
    | 12780c43-084c-4417-aa7f-2e1c505639ab | L3 agent                  | network-node | nova              | :-)   | True           | neutron-l3-agent          |
    | 3f427d52-d2b3-46ee-b372-f83b6f9a3474 | Open vSwitch agent        | compute-node |                   | :-)   | True           | neutron-openvswitch-agent |
    | 5c089ce4-9d6d-4aff-835a-34c3ad1c7b8d | DHCP agent                | network-node | nova              | :-)   | True           | neutron-dhcp-agent        |
    | 68d6e83c-db04-4711-b031-44cf3fb51bb7 | BGP dynamic routing agent | network-node |                   | :-)   | True           | neutron-bgp-dragent       |
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+

As the final step, use the agent id from above and tell the agent that it
should host our BGP speaker:

    # neutron bgp-dragent-speaker-add 68d6e83c-db04-4711-b031-44cf3fb51bb7 bgp1
    Associated BGP speaker bgp1 to the Dynamic Routing agent.

And that will be all. Well, at least from the OpenStack side of things.

# The view from the outside

Of course you will want to verify that everything works fine on your external router, too.
So in our case we have a system running [BIRD](http://bird.network.cz/), which
we configure to be the remote end of the BGP session like this:

    protocol bgp {
      local as 65001;
      neighbor 2001:db8:4321:e0::42 as 65001;
    }

Verify that the session gets established and our prefix is seen as expected:

    bird> show proto bgp1
    name     proto    table    state  since       info
    bgp1     BGP      master   up     12:06:50    Established   
    bird> show route 2001:db8:1234:1::/64
    2001:db8:1234:1::/64 via 2001:db8:4321:2::5 on ens3 [bgp1 12:06:50 from 2001:db8:4321:e0::42] * (100/0) [i]

As extra bonus, verify that the instance we created earlier is reachable from our router:

    router01:~$ ping6 -c3  2001:db8:1234:1:f816:3eff:fecd:6bf4
    PING 2001:db8:1234:1:f816:3eff:fecd:6bf4(2001:db8:1234:1:f816:3eff:fecd:6bf4) 56 data bytes
    64 bytes from 2001:db8:1234:1:f816:3eff:fecd:6bf4: icmp_seq=1 ttl=63 time=1.80 ms
    64 bytes from 2001:db8:1234:1:f816:3eff:fecd:6bf4: icmp_seq=2 ttl=63 time=0.724 ms
    64 bytes from 2001:db8:1234:1:f816:3eff:fecd:6bf4: icmp_seq=3 ttl=63 time=1.04 ms

    --- 2001:db8:1234:1:f816:3eff:fecd:6bf4 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 2000ms
    rtt min/avg/max/mdev = 0.724/1.190/1.803/0.454 ms
    router01:~$ ssh -6 2001:db8:1234:1:f816:3eff:fe58:f80a -l cirros
    cirros@2001:db8:1234:1:f816:3eff:fe58:f80a's password: 
    $ 

# Caveats

In case that things do not work out as expected, here are a couple of things you might check:

- Security group rules for your instances need to be duplicated for IPv6. So even if you did allow
  SSH access via IPv4 (the default selection) already, you will need to add another rule in order
  to allow access via IPv6, too.
- Nova metadata are available via IPv4 only. So if you deploy an instance that has only an IPv6
  address, it will not be able to receive e.g. the SSH keys that should be installed. Either use
  config drive for that or enable a dummy IPv4 network that will allow connectivity to the
  metadata server.
- Most pre-built cloud images are also hardcoded to wait during the boot phase until they have
  received an IPv4 address on their primary interface via DHCP. If you want to avoid this delay
  (which can mean it takes a couple of minutes until you are able to access your instance), you
  either have to build your own image that does some smarter network setup or again you setup
  your dummy IPv4 network.
