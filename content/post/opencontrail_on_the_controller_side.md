+++
date = "2015-10-05T22:40:48+02:00"
title = "Opencontrail on the Controller side"
tags = ["SDN", "NFV", "OpenStack", "Neutron", "OpenContrail"]

+++

*Original post I wrote [here](http://blogs.rdoproject.org/7793/opencontrail-on-the-controller-side)*

In my previous [post](/post/a_journey_of_a_packet_within_opencontrail/) I explained how packets are forwarded from point to point within OpenContrail. We saw the tools available to check what are the routes involved in the forwarding. Last time we focused on the agent side but now we are going to understand on another key component: the controller..

The controller acts as a Route Reflector, announcing routes according to what has been done on the Config/API side and the peering nodes. As we saw in the dataplane post, the controller is using BGP or XMPP in order to exchange routes. XMPP is used between the controllers and the vRouters, BGP is used between the controllers and the gateway routers. One more thing about XMPP, even if it is used for route announcements this is not its only goal. It is used for some other purposes like Service-Chaining, Virtual-Machine related informations, etc.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image02.png" %}}

Taking the same example as previously I will explain how the routes are announced between the controllers and the vRouters and between the controllers and the gateway routers. I also will explain the capability of OpenContrail to extend private network outside the cloud leveraging the L3VPNs.

Ok but, what happens when I boot VM
-----------------------------------

Before continuing to explain the routing aspect within OpenContrail, I think it can be useful to explain what happens when we boot a VM and for what kind of purpose OpenContrail is using XMPP between the controllers and the vRouter.

* When we boot a VM, for example with Nova, the OpenContrail Nova VIF driver[10] asks the vRouter agent for creating a vif interface, thanks to its [API](https://github.com/Juniper/contrail-controller/tree/master/src/vnsw/contrail-vrouter-api). The request is made with all the informations needed to create the interface (name, mac, etc) along with the VM UUID.
* Thanks to this VM UUID, the vRouter agent is going to subscribe to controllers for thatthis particular virtual machine.
* As a result of this subscription, the vRouter agent will receive all the informations (part of the [graph](http://www.if-map.org)) related to this Virtual-Machine (Security-Group, Instance-IP, Routing-Instance, etc.).
* At this point the vRouter agent will be able to announce routes to the controllers about the reachability of our VM.
* The vRouter agent will subscribe with the controller to the Routing-Instance on which our VM is connected, so that it will receive all the routing informations related to this Routing-Instance along with the information on the Virtual-Network.

As usual OpenContrail gives you all the informations about the XMPP exchanges, thanks to the introspect interfaces. Below extracts (formatted) from the introspect interfaces when booting a VM.

**For the agent side :**

http://<agent ip>:8085/Snh_SandeshTraceRequest?x=XmppMessageTrace


```shell
2015-07-25 15:48:41.506 XmppTxStream: Sent xmpp message to: 10.43.91.10 Port 5269 Size: 238 Packet:
<?xml version="1.0" encoding="UTF-8"?>
<iq type="set" from="os2" to="network-control@contrailsystems.com/config">
   <pubsub xmlns="http://jabber.org/protocol/pubsub">
      <subscribe node="virtual-machine:213d8cec-c37d-44f0-809e-51493c9dff84" />
   </pubsub>
</iq>

2015-07-24 15:48:41.510 XmppRxStream: Received xmpp message from: 10.43.91.10 Port 5269 Size: 5229 Packet:
<?xml version="1.0" encoding="UTF-8"?>
<iq type="set" from="network-control@contrailsystems.com" to="default-global-system-config:os2/config">
   <config>
      <update>
         <node type="virtual-machine">
            <name>213d8cec-c37d-44f0-809e-51493c9dff84</name>
            <id-perms>...</id-perms>
            <display-name>213d8cec-c37d-44f0-809e-51493c9dff84</display-name>
         </node>
         <link>
            <node type="virtual-router">
               <name>default-global-system-config:os2</name>
            </node>
            <node type="virtual-machine">
               <name>213d8cec-c37d-44f0-809e-51493c9dff84</name>
            </node>
            <metadata type="virtual-router-virtual-machine" />
         </link>
         <node type="virtual-machine-interface">
          <name>default-domain:demo:55b49e32-b858-48e7-8746-6e09005ecc79</name>
            <virtual-machine-interface-mac-addresses>
               <mac-address>02:55:b4:9e:32:b8</mac-address>
            </virtual-machine-interface-mac-addresses>
            <virtual-machine-interface-device-owner>compute:nova</virtual-machine-interface-device-owner>
      ...
         </node>
         ...
         <node type="instance-ip">
            <name>abc3bd27-d5df-495a-9fb8-dab474470258</name>
            <instance-ip-address>10.0.0.3</instance-ip-address>
            <instance-ip-family>v4</instance-ip-family>
      ...
         </node>
         <link>...</link>
         <node type="virtual-machine-interface-routing-instance">
            <name>attr(default-domain:demo:55b49e32-b858-48e7-8746-6e09005ecc79,default-domain:demo:private:private)</name>
            <value>...</value>
         </node>
         <link>...</link>
      </update>
   </config>
</iq>

...

2015-07-24 15:48:42.766 XmppTxStream: Sent xmpp message to: 10.43.91.10 Port 5269 Size: 871 Packet:
<?xml version="1.0" encoding="UTF-8"?>
<iq type="set" from="os2" to="network-control@contrailsystems.com/bgp-peer" id="pubsub3">
   <pubsub xmlns="http://jabber.org/protocol/pubsub">
      <publish node="1/1/default-domain:demo:private:private/10.0.0.4">
         <item>
            <entry>
               <nlri>
                  <af>1</af>
                  <safi>1</safi>
                  <address>10.0.0.4/32</address>
               </nlri>
               <next-hops>
                  <next-hop>
                     <af>1</af>
                     <address>10.43.91.10</address>
                     <label>19</label>
                     <tunnel-encapsulation-list>
                        <tunnel-encapsulation>gre</tunnel-encapsulation>
                        <tunnel-encapsulation>udp</tunnel-encapsulation>
                     </tunnel-encapsulation-list>
                  </next-hop>
               </next-hops>
               ...
            </entry>
         </item>
      </publish>
   </pubsub>
</iq>

```

**For the controller side :**

http://<controller ip>:8083/Snh_SandeshTraceRequest?x=BgpTraceBuf


```shell
2015-07-25 15:48:42.804 XmppPeerInstanceTrace: XMPP Peer os2:10.43.91.10 in
instance default-domain:demo:private:private : Inet route 10.0.0.4/32
with next-hop 10.43.91.10 and label 19 is enqueued
for add/change

2015-07-25 15:48:42.804 XmppPeerRouteTrace: XMPP Peer os2:10.43.91.10 Insert
new BGP path 10.0.0.4/32 in table default-domain:demo:private:private.inet.0

2015-07-25 15:48:42.804 XmppPeerInstanceTrace: XMPP Peer os2:10.43.91.10 in
instance default-domain:demo:private:private : Evpn route
02:55:b4:9e:32:b8,10.0.0.4/32 with next-hop 10.43.91.10 and
label 20 is enqueued for add/change

```

From High level API resources to Network resources
--------------------------------------------------

To explain how the routing works within OpenContrail let’s take the same example from the previous post, two VMs on two different networks.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image01.png" %}}

In order to interconnect the two networks, we add a virtual router between them. The interesting thing is that, in OpenContrail this router will not be really created. This is just a logical view, even if we can find it requesting the API.

Actually, the virtual router, called Logical-Router in the Opencontrail terminology, will be like translated to some other network resources. There is a special component which handles such translations - *"Schema-Transformer"*. The Schema-Transformer converts high level resources to network resources.

When a virtual network is created at the API side, the Schema-Transformer creates and associates a Routing-Instance to the newly created network. It dynamically allocates a unique  Route-Target (Which acts as a VNI) and associates it to the Routing-Instance.

At the API side we can check the result of the resources created by the Schema-Transformer, thus what is the Routing-Instance for a specific network, for instance for our private network.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image03.png" %}}

and the Route-Target associated…

{{% img-responsive "/images/opencontrail_on_the_controller_side/image01.png" %}}

To summarize, we have two virtual networks, each with a routing-instance with a specific Route-Target :

| Virtual network | Route-Target         |
|-----------------|----------------------|
| private         | target:64512:8000001 |
| service         | target:64512:8000002 |

There is currently three way of defining a [Route-Target](http://tools.ietf.org/html/rfc4364):

* 2 bytes of ASN, 4 bytes of value
* 4 bytes of ASN, 2 bytes of value
* 4 bytes of IP, 2 bytes of value

A route target within OpenContrail is composed of an  ASN and a number higher than 8000000 allocated by the Schema-Transformer and which is of course unique per routing-instance.

Back to our Logical Router, notified by the API, the Schema-Transformer creates a Route-Target and associates it to the newly created Logical Router.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image04.png" %}}

This Route-Target will be associated to the Routing-Instance of each virtual network on which the Logical Router is linked to. Thanks to this Route-Target, shared across the Routing-instances the routes of both network will be leaked at the controller level.

| Virtual network | Route-Target                          |
|-----------------|---------------------------------------|
| private         | target:64512:8000001                  |
|                 | target:64512:8000003 (logical-router) |
| service         | target:64512:8000002                  |
|                 | target:64512:8000003 (logical-router) |

On the API side it will give us two Route-Targets for both Routing-Instances, one for the Virtual Network/Routing-Instance, another one coming from the Logical Router :

{{% img-responsive "/images/opencontrail_on_the_controller_side/image07.png" %}}

As seen in the previous post Routing-Instance are VRFs at vRouter Agent side. Thus a logical router is finally a way to express import/export rules of routes between VRFs, so a way of leaking routes between them.

Controller time !
-----------------

Now that the Schema-Transformer translated the high level resources, we should be able to find them in the controller thanks to the dedicated introspect interface.

- http://<controller ip>:8083/Snh_ShowRoutingInstanceReq

Selecting the private routing instance we get

- The Route-Target imported/exported
- The route tables

{{% img-responsive "/images/opencontrail_on_the_controller_side/image11.png" %}}

We will focus on the route table inet.0 which is the route table for the unicast IPV4 routes.

Clicking on it, we get all the routes details for this route table, ex : from where a route is learned, from os2 and os3, my two compute nodes in this setup as shown below.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image10.png" %}}

I removed some informations from this capture since this is really a wide page, but scrolling to the right we will see that the Next-Hop is indicated as well.

As explain before we can check what are the routes received at the agent level, since it is subscribing to the Routing-Instances. Below the route leaked from the service network :

http://<agent ip>:8085/Snh_SandeshTraceRequest?x=XmppMessageTrace

```shell
2015-07-25 14:29:02.088 XmppRxStream: Received xmpp message from: 10.43.91.10 Port 5269 Size: 1021 Packet:
<?xml version="1.0"?>
<message from="network-control@contrailsystems.com" to="os2/bgp-peer">
<event xmlns="http://jabber.org/protocol/pubsub">
<items node="1/1/default-domain:demo:private:private">
<item id="192.168.0.3/32">
<entry>
    <nlri>...</nlri>
    <next-hops>
        <next-hop>
            <af>1</af>
            <address>10.43.91.12</address>
            <label>16</label>
            <tunnel-encapsulation-list>...</tunnel-encapsulation-list>
        </next-hop>
    </next-hops>
...

```

Routing without virtual router
------------------------------

We just saw that a virtual router is just a logical view. We may want to install routing between our two networks without creating a logical router. For that we have to add the same route target to our both network, let’s say target:64512:444.

Two ways of doing that, the OpenContrail WebUI editing both networks, adding the Route-Target :

{{% img-responsive "/images/opencontrail_on_the_controller_side/image06.png" %}}

or thanks to a command line tool[3], thus via the OpenContrail API :

```shell
$ python add_route_target.py --routing_instance_name \
default-domain:demo:private:private --route_target_number 444 \
--router_asn 64512 --api_server_ip localhost --api_server_port 8082 \
--admin_user admin --admin_password contrail123 --admin_tenant_name admin
```

Currently there is no way to manage Route-Targets with the Neutron API, but there is an on going work through the [BGPVPN service-plugin/extension](https://git.openstack.org/cgit/openstack/networking-bgpvpn).

What about external networks ?
------------------------------

For the external networks, this is almost the same thing. We just need to add a Route-Target to the Routing-Instance of the “external network” on both side, the gateway router and OpenContrail. This Route-Target will take the form of an [extended BGP community](http://tools.ietf.org/html/rfc4360). The gateway routers peering with the controllers using the BGP protocol will be able to import/export routes from/to our “external network”, especially the default route for the OpenContrail side and the floating-ips for the gateway router side.

Assuming we use the Route-Target 64512:555, and we provisioned the gateway router via the OpenContrail WebUI, we can add the Route Target on the routing-instance on the router side. I’m not going to explain here the details on how to setup a gateway router, there is a pretty good explanation [here](http://www.opencontrail.org/how-to-setup-opencontrail-gateway-juniper-mx-cisco-asr-and-software-gw). Below just an extract of a Juniper router configuration.

```shell
# show routing-instances
routing-instances {
    public {
        instance-type vrf;
        interface lt-0/0/0.1;
        vrf-target target:64512:555;
            routing-options {
                    static {
                        route 0.0.0.0/0 next-hop lt-0/0/0.1;
                    }
            }
    }
}
```

Now we have a gateway router with a default route for the public routing-instance, we should be able to find it on the control side.

{{% img-responsive "/images/opencontrail_on_the_controller_side/image05.png" %}}

Associating a floating-ip to a VM, we can check on the router side that the route for the floating-ip is correctly announced.

```shell
> show route table public.inet.0

public.inet.0: 24 destinations, 24 routes (24 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

…

172.16.0.3/32      *[BGP/170] 00:12:24, localpref 200, from 10.43.91.10
                      AS path: ?
                    > via gre.32769, Push 16

...
```

OpenContrail and ExaBGP for fun… troubleshooting
------------------------------------------------

Since the OpenContrail controller is a BGP Speaker we can use the well known BGP Swiss army knife : ExaBGP. Adding ExaBGP as BGP peer to OpenContrail we should be able to dump all the announcements.
Below is an [ExaBGP config file and a python script](https://github.com/safchain/opencontrail_tools/tree/master/exabgp_dump) that aim to dump all the BGP traffic in human readable way.

```shell
group oc {
    router-id 10.43.91.13;
    local-as 64512;
    local-address 10.43.91.13;

    process dump {
        run dump.py;
        encoder json;
        receive {
            parsed;
            update;
        }
    }

    neighbor 10.43.91.10 {
            peer-as 64512;
    }
}
```

```python
#!/usr/bin/python

from datetime import datetime
import json
import os
import pprint
import struct
from sys import stdin, stdout, stderr
import time


def ext_community_to_raw(value):
    raw = ''
    while value:
        b = struct.pack("B", value & 0xff)
        raw = b + raw
        value >>= 8

    if len(raw) < 8:
        raw = '\0' + raw

    return raw


while True:
    try:
        line = stdin.readline().strip()
        msg = json.loads(line)

        try:
            attribute = msg['neighbor']['message']['update']['attribute']
            ext_communities = attribute['extended-community']

            for idx in range(len(ext_communities)):
                ext_community = ext_communities[idx]
                raw = ext_community_to_raw(ext_community)
                if ord(raw[0]) == 0x00 and ord(raw[1]) == 0x02:
                    asn, number = struct.unpack('!HL', raw[2:8])
                    ext_communities[idx] = "target:%d:%d" % (asn, number)
        except:
            pass

        timestamp = datetime.fromtimestamp(msg['time'])
        stderr.write("# %s\n" % timestamp)
        stderr.write(pprint.pformat(msg, width=80) + "\n\n")
    except IOError:
        pass
```

```shell
$ sudo exabgp dump.ini

...
# 2015-08-05 21:26:51
{
  "counter": 2,
  "exabgp": "3.4.8",
  "host": "exa",
  "neighbor": {
    "address": {
      "local": "10.43.91.13",
      "peer": "10.43.91.10"
    },
    "asn": {
      "local": "64512",
      "peer": "64512"
    },
    "ip": "10.43.91.10",
    "message": {
      "update": {
        "announce": {
          "ipv4 mpls-vpn": {
            "10.43.91.10": {
              "172.16.0.3/32": {
                "label": [
                  16
                ],
                "route-distinguisher": "10.43.91.10:2"
              }
            }
          }
        },
        "attribute": {
          "extended-community": [
            "target:64512:555",
            "target:64512:8000004",
            219550481834311682,
            219550481834311693,
            219550481834348681,
            432346663739195393,
            9224775013699817985,
            9255455786153279494
          ],
          "local-preference": 200,
          "origin": "incomplete"
        }
      }
    }
  }
}
...
```

So we see the floating-ip and the Route-Target used for the public network (target:64512:555)

Conclusion
----------

As explained in the previous post, OpenContrail relies on well known protocols and uses them for internally for the control-plane/data-plane. The ability of OpenContrail to use the same protocols internally and externally allows it to be fully integrated in Data-Centers and simplify its integration with access networks.

