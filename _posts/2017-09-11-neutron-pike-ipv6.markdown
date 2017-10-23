---
layout:   post
title:    'Providing globally routed IPv6 adresses to your OpenStack Pike projects'
date:     2017-09-11 13:42:02 +0200
author:   Dr. Jens Harbott
categories: openstack neutron networking ipv6
---

# Note

This is an updated version of an earlier post of mine, taking into account the
changes that have taken place from the Mitaka release, on which the original text
was based, upto the now freshly released Pike version of OpenStack.

The major difference is the use of ``openstack`` commands where available instead
of relying only on python-neutronclient. There are some gaps in functionality
that still need to be closed, though, so the transition is not yet complete.

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
      subnet-pool where the delegation process is not done internally by Neutron, but
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
to select the set of prefixes to be announced within the BGP sessions. All commands shown
in this section will require you to have admin credentials available:

    $ openstack address scope create --share --ip-version 6 address-scope-ip6
    +------------+--------------------------------------+
    | Field      | Value                                |
    +------------+--------------------------------------+
    | id         | ee2ee196-156c-424e-81e5-4d029c66190a |
    | ip_version | 6                                    |
    | name       | address-scope-ip6                    |
    | project_id | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | shared     | True                                 |
    +------------+--------------------------------------+

Next we create the subnetpool that our projects will use to configure their subnets
with:

    $ openstack subnet pool create --address-scope address-scope-ip6 --share --default --pool-prefix 2001:db8:1234::/48 --default-prefix-length 64 --max-prefix-length 64 default-pool-ip6
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | address_scope_id  | ee2ee196-156c-424e-81e5-4d029c66190a |
    | created_at        | 2017-02-24T15:28:27Z                 |
    | default_prefixlen | 64                                   |
    | default_quota     | None                                 |
    | description       |                                      |
    | id                | 4c1661ba-b24c-4fda-8815-3f1fd29281af |
    | ip_version        | 6                                    |
    | is_default        | True                                 |
    | max_prefixlen     | 64                                   |
    | min_prefixlen     | 64                                   |
    | name              | default-pool-ip6                     |
    | prefixes          | 2001:db8:1234::/48                   |
    | project_id        | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | revision_number   | 1                                    |
    | shared            | True                                 |
    | updated_at        | 2017-02-24T15:28:27Z                 |
    +-------------------+--------------------------------------+

Our public network, i.e. the network that the gateway ports of the project routers
will be connected to, needs to be configured with the same address-scope. As the
address scope can only be assigned via subnetpools and not directly, we either have
to assign the IPv6 subnet on the public network from the shared subnetpool we just
created, or we create a second subnetpool containing the specific prefix that we
want to use for our public network:

    $ openstack subnet pool create --address-scope address-scope-ip6 --pool-prefix 2001:db8:4321:42::/64 --default-prefix-length 64 public-pool
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | address_scope_id  | ee2ee196-156c-424e-81e5-4d029c66190a |
    | created_at        | 2017-02-27T10:13:38Z                 |
    | default_prefixlen | 64                                   |
    | default_quota     | None                                 |
    | description       |                                      |
    | id                | 56a1b34b-7e0a-4a76-aac9-8893314ee2a4 |
    | ip_version        | 6                                    |
    | is_default        | False                                |
    | max_prefixlen     | 128                                  |
    | min_prefixlen     | 64                                   |
    | name              | public-pool                          |
    | prefixes          | 2001:db8:4321:42::/64                |
    | project_id        | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | revision_number   | 1                                    |
    | shared            | False                                |
    | updated_at        | 2017-02-27T10:13:38Z                 |
    +-------------------+--------------------------------------+

Now we can create our public network and the IPv6 subnet on it:

    $ openstack network create --provider-network-type flat --provider-physical-network public --external public
    <output skipped>
    $ openstack subnet create --ip-version 6 --subnet-pool public-pool --network public public-ip6
    +-------------------+----------------------------------------------------------+
    | Field             | Value                                                    |
    +-------------------+----------------------------------------------------------+
    | allocation_pools  | 2001:db8:4321:42::2-2001:db8:4321:42:ffff:ffff:ffff:ffff |
    | cidr              | 2001:db8:4321:42::/64                                    |
    | created_at        | 2017-02-27T10:23:00Z                                     |
    | description       |                                                          |
    | dns_nameservers   |                                                          |
    | enable_dhcp       | True                                                     |
    | gateway_ip        | 2001:db8:4321:42::1                                      |
    | host_routes       |                                                          |
    | id                | 77551166-fb97-4ea5-912a-c17c75a05eda                     |
    | ip_version        | 6                                                        |
    | ipv6_address_mode | None                                                     |
    | ipv6_ra_mode      | None                                                     |
    | name              | public-ip6                                               |
    | network_id        | 28c08355-cb8f-4b1b-b5fd-f5442e531b28                     |
    | project_id        | 6de6f29dcf904ab8a12e8ca558f532e9                         |
    | revision_number   | 2                                                        |
    | segment_id        | None                                                     |
    | service_types     |                                                          |
    | subnetpool_id     | 56a1b34b-7e0a-4a76-aac9-8893314ee2a4                     |
    | updated_at        | 2017-02-27T10:23:00Z                                     |
    +-------------------+----------------------------------------------------------+

Verify that the address-scope for IPv6 indeed got set for this network via creating the subnet from the
proper subnetpool:

    $ openstack network show public
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | UP                                   |
    | availability_zone_hints   |                                      |
    | availability_zones        |                                      |
    | created_at                | 2017-02-27T10:21:04Z                 |
    | description               |                                      |
    | dns_domain                | None                                 |
    | id                        | 28c08355-cb8f-4b1b-b5fd-f5442e531b28 |
    | ipv4_address_scope        | None                                 |
    | ipv6_address_scope        | ee2ee196-156c-424e-81e5-4d029c66190a |
    | is_default                | False                                |
    | mtu                       | 1500                                 |
    | name                      | public                               |
    | port_security_enabled     | True                                 |
    | project_id                | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | provider:network_type     | flat                                 |
    | provider:physical_network | external                             |
    | provider:segmentation_id  | None                                 |
    | qos_policy_id             | None                                 |
    | revision_number           | 6                                    |
    | router:external           | External                             |
    | segments                  | None                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   | 77551166-fb97-4ea5-912a-c17c75a05eda |
    | updated_at                | 2017-02-27T10:23:00Z                 |
    +---------------------------+--------------------------------------+

# View from the other side

We have now prepared everything on the admin side of things, so let our users start to configure their networking.
The commands in this section are meant to be executed with user credentials:

    $ openstack network create mynet
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | UP                                   |
    | availability_zone_hints   |                                      |
    | availability_zones        |                                      |
    | created_at                | 2017-02-27T10:26:22Z                 |
    | description               |                                      |
    | dns_domain                | None                                 |
    | id                        | 1f20da97-ddd4-40f8-b8d3-6321de8671a0 |
    | ipv4_address_scope        | None                                 |
    | ipv6_address_scope        | None                                 |
    | is_default                | None                                 |
    | mtu                       | 1450                                 |
    | name                      | mynet                                |
    | port_security_enabled     | True                                 |
    | project_id                | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | provider:network_type     | None                                 |
    | provider:physical_network | None                                 |
    | provider:segmentation_id  | None                                 |
    | qos_policy_id             | None                                 |
    | revision_number           | 3                                    |
    | router:external           | Internal                             |
    | segments                  | None                                 |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | updated_at                | 2017-02-27T10:26:22Z                 |
    +---------------------------+--------------------------------------+

In order to add some IPv6 prefix, simply request one from the default pool:

    $ openstack subnet create --ip-version 6 --use-default-subnet-pool --ipv6-address-mode slaac --ipv6-ra-mode slaac --network mynet mysubnet
    +------------------------+--------------------------------------------------------+
    | Field                  | Value                                                  |
    +------------------------+--------------------------------------------------------+
    | allocation_pools       | 2001:db8:1234:1::2-2001:db8:1234:1:ffff:ffff:ffff:ffff |
    | cidr                   | 2001:db8:1234:1::/64                                   |
    | created_at             | 2017-02-27T11:14:23Z                                   |
    | description            |                                                        |
    | dns_nameservers        |                                                        |
    | enable_dhcp            | True                                                   |
    | gateway_ip             | 2001:db8:1234:1::1                                     |
    | host_routes            |                                                        |
    | id                     | 193f7620-6c4c-4adc-9bb5-ff73c9b08d59                   |
    | ip_version             | 6                                                      |
    | ipv6_address_mode      | slaac                                                  |
    | ipv6_ra_mode           | slaac                                                  |
    | name                   | mysubnet                                               |
    | network_id             | 1f20da97-ddd4-40f8-b8d3-6321de8671a0                   |
    | project_id             | 6de6f29dcf904ab8a12e8ca558f532e9                       |
    | revision_number        | 2                                                      |
    | segment_id             | None                                                   |
    | service_types          |                                                        |
    | subnetpool_id          | 4c1661ba-b24c-4fda-8815-3f1fd29281af                   |
    | updated_at             | 2017-02-27T11:14:23Z                                   |
    | use_default_subnetpool | true                                                   |
    +------------------------+--------------------------------------------------------+

Note that the ''--use-default-subnet-pool'' option is broken in OSC <= 3.9.0, if you are seeing an error like

    HttpException: Bad Request, Bad subnets request: a subnetpool must be specified in the absence of a cidr.

then instead specify the subnet pool explicitly with ''--subnet-pool default-pool-ip6''.

Next we want outside connectivity, so we create a router, add an interface into our project net and set the gateway to be
the public network:

    $ openstack router create router1
    +-------------------------+--------------------------------------+
    | Field                   | Value                                |
    +-------------------------+--------------------------------------+
    | admin_state_up          | UP                                   |
    | availability_zone_hints |                                      |
    | availability_zones      |                                      |
    | created_at              | 2017-02-27T12:59:06Z                 |
    | description             |                                      |
    | distributed             | False                                |
    | external_gateway_info   | None                                 |
    | flavor_id               | None                                 |
    | ha                      | False                                |
    | id                      | d2db0603-fda2-4305-a1de-e793a36c0770 |
    | name                    | router1                              |
    | project_id              | 6de6f29dcf904ab8a12e8ca558f532e9     |
    | revision_number         | None                                 |
    | routes                  |                                      |
    | status                  | ACTIVE                               |
    | updated_at              | 2017-02-27T12:59:06Z                 |
    +-------------------------+--------------------------------------+
    $ openstack router add subnet router1 mysubnet
    $ openstack router set --external-gateway public router1

Finally we boot an instance and verify that it gets an IPv6 address assigned:

    $ openstack server create --flavor c1 --image cirros vm1
    $ openstack server list
    +--------------------------------------+------+--------+-------------+--------------------------------------------+------------+
    | ID                                   | Name | Status | Power State | Networks                                   | Image Name |
    +--------------------------------------+------+--------+-------------+--------------------------------------------+------------+
    | 17b2ac04-9a17-45ff-be30-401aa8331a66 | vm1  | ACTIVE | Running     | mynet=2001:db8:1234:1:f816:3eff:fe53:f89e  | cirros     |
    +--------------------------------------+------+--------+-------------+--------------------------------------------+------------+
    $ openstack console log show vm1 | grep -A1 -B1 2001
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
    # Important thing is you add the BgpPlugin to the list
    service_plugins = neutron_dynamic_routing.services.bgp.bgp_plugin.BgpPlugin,neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
    # In case you run into issues, this will also be helpful
    debug = true

You need to restart your ``neutron-server`` process(es) in order activate the plugin.
Now we can create our first BGP speaker. We set the IP version to 6, select some
private ASN that we can use for our POC and disable advertising floating IPs, as we
do not have these for IPv6 anyway:

    # openstack bgp speaker create --ip-version 6 --local-as 65001 --advertise-floating-ip-host-routes false bgp1
    +-----------------------------------+--------------------------------------+
    | Field                             | Value                                |
    +-----------------------------------+--------------------------------------+
    | advertise_floating_ip_host_routes | False                                |
    | advertise_tenant_networks         | True                                 |
    | id                                | b9547458-7bdd-4738-bd57-a985055fc59c |
    | ip_version                        | 6                                    |
    | local_as                          | 65001                                |
    | name                              | bgp1                                 |
    | networks                          | []                                   |
    | peers                             | []                                   |
    | project_id                        | c67d6ba16ea2484597061245e5258c1e     |
    | tenant_id                         | c67d6ba16ea2484597061245e5258c1e     |
    +-----------------------------------+--------------------------------------+

We add our public network to this speaker, indicating that we want to advertise all those
tenant networks here, which have a router with the public network as gateway. We also
verify that our tenant network gets listed for advertisement:

    # openstack bgp speaker add network bgp1 public
    # openstack bgp speaker list advertised routes bgp1
    +----+--------------------+---------------------+
    | ID | Destination        | Nexthop             |
    +----+--------------------+---------------------+
    |    | 2001:db8:1234::/64 | 2001:db8:4321:42::c |
    +----+--------------------+---------------------+

Assume our external router has the address 2001:db8:4321:e0::1, we configure it as
BGP peer and add it to our BGP speaker:

    # openstack bgp peer create --peer-ip 2001:db8:4321:e0::1 --remote-as 65001 bgp-peer1
    +------------+--------------------------------------+
    | Field      | Value                                |
    +------------+--------------------------------------+
    | auth_type  | none                                 |
    | id         | 0183260e-b1d0-40ae-994f-075668b99676 |
    | name       | bgp-peer1                            |
    | peer_ip    | 2001:db8:4321:e0::1                  |
    | project_id | c67d6ba16ea2484597061245e5258c1e     |
    | remote_as  | 65001                                |
    | tenant_id  | c67d6ba16ea2484597061245e5258c1e     |
    +------------+--------------------------------------+
    # openstack bgp speaker add peer bgp1 bgp-peer1
    #

# The missing link

So all that it left now is to setup the agent that will start the BGP session and
advertise the prefixes to the outside world. On the network node, install the
necessary packages, e.g. for Ubuntu Xenial:

    # apt install neutron-bgp-dragent python-ryu

This will also add a configuration file at ``/etc/neutron/bgp_dragent.ini`` which
needs some tuning:

    [BGP]
    # BGP speaker driver class to be instantiated. (string value)
    bgp_speaker_driver = neutron_dynamic_routing.services.bgp.agent.driver.ryu.driver.RyuBgpDriver

    # 32-bit BGP identifier, typically an IPv4 address owned by the system running
    # the BGP DrAgent. (string value)
    bgp_router_id = 10.11.12.13

Again you will have to restart the agent in order for it to pick up the new
configuration. If all goes well, the agent should register itself in your setup:

    # openstack network agent list
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+
    | ID                                   | Agent Type                | Host         | Availability Zone | Alive | State          | Binary                    |
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+
    | 12780c43-084c-4417-aa7f-2e1c505639ab | L3 agent                  | network-node | nova              | :-)   | True           | neutron-l3-agent          |
    | 3f427d52-d2b3-46ee-b372-f83b6f9a3474 | Open vSwitch agent        | compute-node |                   | :-)   | True           | neutron-openvswitch-agent |
    | 5c089ce4-9d6d-4aff-835a-34c3ad1c7b8d | DHCP agent                | network-node | nova              | :-)   | True           | neutron-dhcp-agent        |
    | 68d6e83c-db04-4711-b031-44cf3fb51bb7 | BGP dynamic routing agent | network-node |                   | :-)   | True           | neutron-bgp-dragent       |
    +--------------------------------------+---------------------------+--------------+-------------------+-------+----------------+---------------------------+

As the final step, use the agent id from above and tell the agent that it
should host our BGP speaker:

    # neutron bgp-dragent-speaker-add 68d6e83c-db04-4711-b031-44cf3fb51bb7 bgp1
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
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
  config drive for that or create a dummy IPv4 subnet that will allow connectivity between your
  instance and the metadata server.
- Most pre-built cloud images are also hardcoded to wait during the boot phase until they have
  received an IPv4 address on their primary interface via DHCP. If you want to avoid this delay
  (which can mean it takes a couple of minutes until you are able to access your instance), you
  either have to build your own image that does some smarter network setup or again you setup
  your dummy IPv4 network.
- Some of the `openstack` commands being used in the examples have not yet been released at the
  time of this writing, see in particular this review: https://review.openstack.org/340763
