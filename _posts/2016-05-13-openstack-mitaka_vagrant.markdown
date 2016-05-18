---
layout:   post
title:    'Cooking some "Mitaka" flavoured OpenStack on your local machine'
date:     2016-05-13 16:48:02 +0200
author:   Jan Klare
categories: openstack chef
published: false
---

# Cooking some "Mitaka" flavoured OpenStack on your local machine

Hi everybody and welcome to the OpenStack-Chef kitchen. Today we are going to
cook ourselvs some nice and tasty OpenStack Mitaka. And since we are not any
ordinary boring one-node-only cooking show, we will cook a whole cluster of it
to satisfy the cravings of all your developers at once. Lets get started then!

## The ingredients

To get started with this, your kitchen should contain at least 16GB of ram, 4
cores and some harddisk to play on. The kitchen I am cooking in today is a
simple macbook pro with 16GB of ram and a 3,1 GHz i7 from early 2015. If your
kitchen lacks some of these equipments, you can still try, but there is no
guarantee that you will be able to cook the same cluster we are going for
without running out of resources.

We will be using [virtualbox](https://www.virtualbox.org/) and
[vagrant](https://www.vagrantup.com/) and before you start you should get and
install them according to the guides for your platform.

In addition to the kitchen itself we will need a whole bunch of cookbooks, but
luckily you can get all of them at a one-stop-berkself directly from the
[openstack-chef-repo](https://github.com/openstack/openstack-chef-repo). Just
pull the repo to your favourite git location.

Since we will be using a lot of the standard tools in store the chef business,
you should either have all the needed gems already bundled up somewhere, or you
should download and install the fitting
[chefdk](https://downloads.chef.io/chef-dk/) with the version 0.9.0. I will
recommend to go with the chefdk 0.9.0, since we used it too and it worked. In
case you wonder why we are not using the most recent version: We tested with
0.9.0 during the whole development and it still works and somebody said "never
change a winning team".

If you have prepared all the things mentioned above and already some appetite
for a nice and tasty OpenStack, you should go ahead and continue with the next
steps.

## Mise en Place

To get started you should cd to the openstack-chef-repo you pulled before and
have a quick look at some of the core documents we will use during the actual
cooking.

### The Rakefile

The first and most mysterious thing here is the
[Rakefile](https://github.com/openstack/openstack-chef-repo/blob/master/Rakefile),
which contains a lot of useful methods and has them already bundled in tasks to
deploy and test different scenarios of OpenStack. Of course you can go ahead and
read the whole file (which might even be a good idea in case you want to
continue working with this chef-repo), but for today we will just use three of
these tasks:

1. [berks_vendor](https://github.com/openstack/openstack-chef-repo/blob/master/Rakefile#L27-L30):
As stated in the description, this task will pull in all of the needed cookbooks
for today's cooking session. It will read the
[Berksfile](https://github.com/openstack/openstack-chef-repo/blob/master/Berksfile),
use [berkshelf](http://berkshelf.com/) to resolve all the dependencies and
download all cookbooks to the cookbooks folder inside the openstack-chef-repo.

2. [multi_node](https://github.com/openstack/openstack-chef-repo/blob/master/Rakefile#L45-L48):
This task will automate all the cooking for you and the only thing you have to
do during this, is take a look at how fast this one will build a local
three-node OpenStack cluster. It will read the
[vagrant_linux.rb](https://github.com/openstack/openstack-chef-repo/blob/master/vagrant_linux.rb)
and pull in the needed vagrant box. This can be either ubuntu 14.04 or centos
7.2, which is completely up to you and switchable by the environment variable
['REPO_OS'](https://github.com/openstack/openstack-chef-repo/blob/master/vagrant_linux.rb#L12).
In addition to that, it will also use the
[multi-node.rb](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb)
which specifies the exact configuration of all virtual machines
([controller_config](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb#L4-19)
and
[compute_config](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb#L35-45))
and how these should be created
with [chef-provisioning](https://github.com/chef/chef-provisioning). It will
create one
[controller-node](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb#L25)
with the role
['multi-node-controller'](https://github.com/openstack/openstack-chef-repo/blob/master/roles/multi-node-controller.json)
and two
[compute-nodes](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb#L48)
with the role
['multi-node-compute'](https://github.com/openstack/openstack-chef-repo/blob/master/roles/multi-node-compute.json).
So after running this, we will have a total of three nodes, which will be
controller, compute1 and compute2.

3. [clean](https://github.com/openstack/openstack-chef-repo/blob/master/Rakefile#L50-L51):
This task is pretty straightforwand and well described with "Blow everything
away". You can and should use it in case you get bored after you have seen the
whole cluster work or in case you get stuck somewhere and want to start fresh.
Since it says "Blow everything away", you really will need to start at the
beginning by vendoring your cookbooks (1).

### The environment
As you might have seen already in the
[multi-node.rb](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb),
we will also use a [distribution specific
environment](https://github.com/openstack/openstack-chef-repo/blob/master/multi-node.rb#L21-22),
to get some additional flavor into our cluster. Since the two environment files
are very similar, we will just look at the [ubuntu
one](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json)
for now and you should be
able to walk yourself through the [centos
one](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-centos7.json)
if you need it.

#### apache2 and apt
Starting from the
[top](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L8-10),
we will reset the `node['apache']['listen']` to `[]` in order to not use the
[default](https://github.com/svanzoest-cookbooks/apache2/blob/v3.2.2/attributes/default.rb#L279)
anymore, since this would interfere with our configuration.
Additionally we will configure
[apt](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L11-13)
to run a full update of its sources during the compile time, so we can install
the newest stuff right from the beginning (and even during the compile_time).

#### And now for the more interesting part: OpenStack OpenStack

Since we want to deploy OpenStack with the
[openstack-chef-cookbooks](https://github.com/openstack?utf8=%E2%9C%93&query=cookbook-openstack-),
we will need to pass some attributes to these, to align them with all our
expectations for a three node cluster.

#### sysctl
The first thing we want to do here, is to allow all nodes to
[forward](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L17-19)
network traffic. This is needed, since we want to run our routers and dhcp
namespaces on the controller and connect them via openvswitch (ovs) bridges to
the instances running on the two compute nodes.

#### endpoints
To actually allow all the OpenStack services to talk to each other, either
via the [message
queue](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L21)
or directly via the
[APIs](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L22-61),
we need to define the endpoints we want to use. Since all of the APIs and the
message queue (mq) will be running on our controller node, we will configure one of
its IP-addresses ('192.168.101.60') as the 'host' attribute for the endpoints
and the mq. With this configuration, all of the OpenStack service APIs will be
reachable via their default ports (e.g. [9696 for
neutron](https://github.com/openstack/cookbook-openstack-network/blob/master/attributes/default.rb#L34))
on the address '192.168.101.60' (e.g. '192.168.101.60:9696' for neutron).

#### binding services
Right below the endpoint setting, we see a whole block that looks quite similar
to the endpoint one, but is called
['bind_service'](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L62-82).
In addition to the endpoints where the service will be reachable, we also need
to define where the actual service should be listening. You might think that
this is the exact same thing, but it's not. In the most production environments
you will need additional proxies like 'haproxy' or 'apache' right in front of
your APIs for security, filtering, threading and HA. This said, you the endpoint
where your API is reachable might in fact be an 'apache' or 'haproxy',
listening on a completely different ip and port than your actual OpenStack
service is. During the design of the cookbooks we decided to bind all of the
services by default to '127.0.0.1' so we have some security by default and do
not make them world accessible. In our scenario today however, we need them to
be accessible by our compute nodes and from the outside of the vagrant boxes
(since we maybe want to test the cli tools on our local machine against the
APIs), and will therefore bind them to '0.0.0.0' to avoid a more complex
configuration. This will make them accessible on their default ports via all
IP addresses assigned to the controller node.  The next important setting in our
environment is the detailed and attribute driven configuration of the networking
service neutron.

#### Neutron ml2 plugin
In the [first
section](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L90-99)
we configure the ml2 plugin we want to use for our virtual networks. In this
case we want to go with the default ml2 plugin using vxlan as the overlay to
seperate tenant networks.

#### Neutron ovs tunnel interface
To allow the actual traffic to flow between instances and router/dhcp
namespaces, we need to additionally specify the interface we want to create our
overlay vxlan ovs bridge on. For our scenario this will be the
['eth1'](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L90-99)
interface on the controller and compute nodes. The actual ovs bridge
configuration will be done with the [example
recipe](https://github.com/openstack/cookbook-openstack-network/blob/master/recipes/_bridge_config_example.rb)
from the network cookbook.

#### neutron.conf and ml2_plugin.ini
In the following and last networking section, we are setting some configuration
parameters that will directly go into the neutron.conf. In our scenario we want
to use the neutron-l3-agent and therefore need to enable the [service_plugin
'router'](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L103).
We also need to specify where the neutron-openvswitch-agent running on our
compute nodes can find the
[mq](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L105)
to talk to the other Neutron agents. And since we want to use vxlan as the
default for our [tenant networks](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L106), we need to specify this as well.

#### upload default cirros image
To be able to instantly start some instances after we have the cluster up and
running, we are also enabling the [image
upload](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L109-111)
of a simple cirros image.

#### nova.conf
The last section in our environment is dedicated to configuring Nova. Since
we will be running our cluster on top of virtualisation, we do not want to use
the default 'virt_type' kvm, but rather go with
[qemu](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L115).
The last option for the 'oslo_messaging_rabbit' section enables nova-compute to
talk to the
[mq](https://github.com/openstack/openstack-chef-repo/blob/master/environments/multi-node-ubuntu14.json#L117)
and all of the other nova service (same as for the neutron-openvswitch-agent).

I guess we have now spend enough time with our *Mise en Place* and should start
the actual cooking, people are getting hungry.

## Cooking like a boss

As all of you chefs might now, if you have done a good *Mise en Place*, cooking
becomes a breeze. The only thing we need to do now to get things started is
getting all our cookbooks with the 'berks_vendor' task (1) and run the
'multi_node' task (2) from the Rakefile mentioned above. If you are using
chefdk you can do this by running:

```
chef exec rake berks_vendor
chef exec rake multi_node
```

> ### NOTICE!
> If your first chef run fails while installing the package "cinder-common",
> you are probably on a mac and there seems to be strange issue with handing
> over the locales during a chef run. Just start the run again with:

> ```chef exec rake multi_node```

You should now get yourself a coffee and maybe even some fresh air, since this
will take a while.

## Serving the Mitaka flavored OpenStack cluster

After roundabout 15-20 minutes, depending on your kitchen hardware, you will
have a full OpenStack Mitaka ready for consumption. Now lets dig into it!

> ### NOTICE!
> At the time of writing, there is a rather unpleasent bug in the startup of
> libvirt-bin, since the default logging service virtlogd seems to be started but
> instantly crashes. The bug is documented on
> [launchpad](https://bugs.launchpad.net/ubuntu/+source/libvirt/+bug/1577455) and
> can simply be fixed by starting the service virtlogd manually or restarting the
> whole compute nodes.  To start the service manually ssh to the compute1 and
> compute2 and run:

> ```sudo service virtlogd start```

> After that you should be good to go.


Most people like to start with the good looking stuff, so we will go ahead and
navigate to the
[dashboard](http://docs.openstack.org/user-guide/dashboard_log_in.html), which
should be accessible on
[https://localhost:9443](https://localhost:9443). You can log in as the 'admin'
user with the password 'mypass'.

You should really enjoy this part a bit longer, maybe [create some
networks, routers](http://docs.openstack.org/user-guide/dashboard_create_networks.html),
and even [launch some
instances](http://docs.openstack.org/user-guide/dashboard_launch_instances.html)
directly off the cirros images that was automagically uploaded.

If you enjoyed the dashboard, you maybe want to dig a bit deeper and try to work
with the [command line clients](http://docs.openstack.org/user-guide/cli.html)
directly from the controller. To do so, you should go back to your
openstack-chef-repo and navigate to the subfolder 'vms'. Inside of that folder
you can use vagrant to directly ssh to the controller or one of the compute
nodes like this:

```
# ssh to controller
vagrant ssh controller
# ssh to the first compute node
vagrant ssh compute1
```

Once you are on the controller, you should become 'root' and load the
environment variables from the provided openrc file in /root/openrc like this:

```
# become root
sudo -i
# load openrc environment variables
. /root/openrc
```

Since all of the python clients you need to talk to the OpenStack APIs were
already installed during the deployment, you can now go ahead and use them to
either do the same things you did on the dashboard above ([create networks,
routers](http://docs.openstack.org/user-guide/cli_create_and_manage_networks.html)
and [launch some
instances](http://docs.openstack.org/user-guide/cli_launch_instances.html)) or
try something new and play a little with
[heat](http://docs.openstack.org/user-guide/cli_create_and_manage_stacks.html),
since chefs usually love hot cooking.

## Cleaning up
As soon as you decide that you had enough OpenStack for today, you can exit the
controller node, navigate back to the openstack-chef-repo root directory and
clean up your whole kitchen with the 'clean' task (3) from the Rakefile
mentioned at the beginning.

```
chef exec rake clean
```

I think thats it for today, if you have any questions regarding this setup, come
and find me and the other openstack chef core reviewers at the irc channel on
freenode #openstack-chef.

## Happy cooking!
