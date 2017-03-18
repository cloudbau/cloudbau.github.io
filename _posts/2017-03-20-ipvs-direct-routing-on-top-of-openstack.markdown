---
layout:   post
title:    'IPVS direct routing on top of OpenStack'
date:     2017-03-20 12:00:00 +0200
author:   Jan Klare
categories: openstack loadbalancing networking ipvs
---

# Motivation

So, why am I writing about such a super old
technology that comes
from a world before anybody even started to think about OpenStack? The answer is
pretty simple: "Customers love technology that has already proven to work and is
easy to configure!". Ok, I initially thought
[IPVS](http://www.linuxvirtualserver.org/software/ipvs.html) itself might not be
a very good fit for a loadbalancer on top of OpenStack and the documentation on
how to configure and run it is pretty poor and super old. Nevertheless the other
options to achieve loadbalancing for non-http traffic without loosing the
original source IP address are not that numerous. The only real option I found
was the
[proxy-protocol](http://www.haproxy.org/download/1.8/doc/proxy-protocol.txt)
which is also used for the [AWS Elastic Load
Balancing](https://aws.amazon.com/elasticloadbalancing/). I had a quick look
into that and might try to use this at a later point in time, but at first
sight, it did not seem as simple and as straight forward as ipvs for the task.

# The Task

So what is it we were trying to achieve? We need a loadbalancer that is able
to handover all client traffic without loosing the original source IP address
and also allow the "real servers" that handle the load to answer the client
directly in order to avoid the bottleneck of returning traffic via the loadbalancer
itself. So the traffic flow should look similar to this:

  client -> loadbalancer -> real-server -> client

A more detailed description of LVS/DR including a nice graphic can be found
[here](http://www.linuxvirtualserver.org/VS-DRouting.html).

# The Testing Setup

To get started with the technology we created a quite simple setup that looked
like this:

```
server-1 (ipvs-loadbalancer):
  eth0: 203.0.113.3/27
        203.0.113.2/32
server-2 (apache-server-1)
  eth0: 203.0.113.4/27
server-3 (apache-server-2)
  eth0: 203.0.113.5/27
```

In our testing setup we directly attached the public IPv4 addresses out of a
routed provider network to our servers without any NAT or router namespace used.

If you do this with NAT and a router namespace and only have private IP
addresses on your server, you need to add some more configuration regarding the
allowed address pairs. Additionally you will probably not get the intended
result, since all traffic will pass through one router namespace outside of your
setup, which creates exactly the bottleneck we were initially trying to avoid
here.

All servers were booted with a recent Ubuntu Xenial cloud image on an
OpenStack "Newton" (we also tried this on an Icehouse cluster to make
sure the release does not matter here). The gateway for all servers was an
external router (203.0.113.1) and the traffic flow we tried to achieve was:

```
client -> 203.0.113.2/32 -> (server-2 or server-3) -> client
```

Obviously for this to work we needed to achieve at least three things:

1. traffic that is directed to server-1 on 203.0.113.2/32 needs to be
redirected to server-2 or server-3 
2. server-2 and server-3 need to accept the packets for 203.0.113.2/32
3. after accepting the client request, server-2 and server-3 need to answer
directly to the client using the original target IP 203.0.113.2/32 as the
source IP for all packets (otherwise the client will just drop them).

The first part is pretty easy. We installed the "ipvsadm" package on server-1 and
followed the manpage on how to create a ipvs virtual server including the "real"
target servers.

```
ipvsadm -A -t 203.0.113.2:80 -s rr
ipvsadm -a -t 203.0.113.2:80 -r 203.0.113.4:80 -g
ipvsadm -a -t 203.0.113.2:80 -r 203.0.113.5:80 -g
echo 1 > /proc/sys/net/ipv4/ip_forward #to enable forwarding
```

This created a configuration that looked like this:

```
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  203.0.113.2:http rr
  -> 203.0.113.4:http            Route   1      0          0
  -> 203.0.113.5:http            Route   1      0          0
```

It is quite important that the "Forward" method here is "Route" since we want to
avoid sending the traffic back via the loadbalancer, but rather let the "real
servers" answer directly.

To allow the server-1 to forward/redirect traffic with the original source IP
from the client, we added an "allowed-address" range on this port (eth0
in our example).

```
openstack port set $ID_OF_ETH0_FROM_SERVER_1 --allowed-address ip-address=0.0.0.0/0
```

Just to check if everything worked so far, we started sending some requests to
203.0.113.2:80 at this point and were able to capture them with tcpdump on eth0
of server-2 and server-3 (of course our security groups were configured to allow
ingress traffic on port 80 for all servers). Initially the requests/packets were
not accepted by server-2 and server-3, since they were not targeted to any
address they owned.

To make the servers accept these packages, we created a dummy-interface
"eth-vip" on server-2 and server-3 were we could add our virtual ip
203.0.113.2/32.

```
ip link add eth-vip type dummy
ip link set eth-vip up
```

Initially we just added the address and wondered why basically everything
stopped working. Thinking about it, the answer here is pretty simple
and more related to basic networking principles than the setup itself. If you
configure the same IP address on multiple servers in the same layer2 domain, arp
kills you network. There is a very very long discussion/documentation on how to
avoid this for IPVS
[here](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.arp_problem.html).
We tried different methods mentioned there. In the end we decided to go with
two very simple arptable entries on server-2 and server-3 since they seemed like
the "cleanest" solution to the problem.

```
arptables -A INPUT -d 203.0.113.2 -j DROP
arptables -A OUTPUT -s 203.0.113.2 -j DROP
ip a add 203.0.113.2/32 dev eth-vip
```

It is probably also important to mention here, that you need at least two
addresses on eth0 of your server-1 in the same network to still allow
proper arp between all servers, since server-2 and server-3 will not send back
any arp responses coming from 203.0.113.2 to server-1 after they have this
address configured themselves.

After that was all done, the only thing we had to do was adding some more
"allowed-addresses" to the eth0 ports of server-2 and server-3 since they 
now started answering client requests they recieved on 203.0.113.2 from this
address.

```
openstack port set $ID_OF_ETH0_FROM_SERVER_2/3 --allowed-address ip-address=203.0.113.2/32
```

Of course the last steps were to actually install an apache2 on server-2 and
server-3 that served some answers to our requests, but I will skip the guide for
this part here.

# Future Work

Since this was only our first testing setup with this technology, we are still
investigating if it fits all our needs and scales/performs as well as we
currently believe. In addition to that, we will also look a bit deeper into the
proxy-protocol mentioned in the beginning to see if this is a better fit here.
