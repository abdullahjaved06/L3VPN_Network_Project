# L3VPN Project

In this second project, you will configure a network to provide a L3VPN service. You will need to provide isolation between the different clients you provision, while ensuring that the different sites of clients are connected. In practice, you will have two clients: Apple (A) and Bad (B). Each client will have a varying number of sites.

## First steps

To do a L3VPN, you will need to enable the BGP frrouting daemon. 
You can check [the first practical](../00-tp-isis/README.md#introduction-to-ffrouting) to learn how to do that.

Then, configure IS-IS in a similar manner to the first practical to enable intra-domain connectivity.

All the documentation needed for this project can be found on the [BGP page of frrouting](https://docs.frrouting.org/en/latest/bgp.html), especially the L3VPN section, as well as the [IS-IS page of frrouting](https://docs.frrouting.org/en/latest/isisd.html), which you will use to configure SRv6.

## Project overview

For this project, here are the different steps that you need to perform:

- Create VRFs and assigning interfaces
- Configure SRv6
- Configure iBGP sessions
- Configure eBGP sessions
- Export routes

## Create VRFs and assigning interfaces

First, make sure that the VRF module is loaded by running `sudo modprobe vrf` on your host computer.

On each PE, create one VRF per client and assign the interface towards the CE router to its corresponding VRF.

The following resource will explain you how to create a VRF and how to assign an interface to a VRF: https://docs.kernel.org/networking/vrf.html

## Configure SRv6

In this project, we will use SRv6 to provide the VRF lookup mechanism. 

First, specify the locator for each node. [The official documentation](https://docs.frrouting.org/en/latest/zebra.html#segment-routing-ipv6) explains how to do so. Use a simple scheme to assign a locator to each node.

Then, once a locator has been assigned to each node, you will need to propagate this locator within your network, to ensure that every node knows for each locator the node responsible for this locator. Fortunately, frrouting allows to distribute SRv6 locator information with IS-IS. We thus ask you to configure IS-IS to disseminate locator information.

**Important note**: to use SRv6, you need to set `net.vrf.strict_mode=1`, otherwise SRv6 routes will be rejected by the kernel. You will also need to enable the `net.ipv6.conf.all.seg6_enabled` sysctl setting. A description of this setting can be found on https://segment-routing.org/index.php/Implementation/Configuration

## Configure iBGP sessions

After this, you will need to configure iBGP sessions to ensure that BGP routes are exchanges between routers inside your network. 

This resource explains how BGP can be configured to create (internal) peering session: https://gitlab.ethz.ch/nsg/public/comm-net-2022-routing-project/-/wikis/2.-Tutorial/2.5-Configuring-IP-routers/2.5.5-Configure-BGP

As we are working on iBGP, make sure that the remote-as is identical to the AS you chose for the L3VPN network. Also, don't forget to configure the BGP session to use next-hop-self, as well as using the loopback address for the session (update-source option).

Before moving to the next step, ensure that iBGP session are indeed working by using `show bgp summary`. You should see each neighbor as "UP"

## Configure eBGP sessions

You will first need to manually assign IPv6 addresses on both the interface shared between peers of an eBGP session. Choose these addresses as you please, then make sure that the peer is reachable from each host before proceeding further.

### On CE routers

Configure CE routers to create peering sessions with the provider, in a similar manner to what is described in the [iBGP section](#configure-ibgp-sessions). Note however that you will need to set a different remote-as when configuring BGP, as we are now dealing with eBGP. In addition, configure the routers to announce a prefix. You can choose how prefixes are assigned to clients/sites, but try to use once again a simple scheme.

**Important note**: the BGP implementation of frrouting refuses to announce a prefix if a route towards this prefix does not exist. To force the announcement of a prefix, you can add the vtysh command `ipv6 route $PREFIX Null0` to simulate the fact that the prefix is connected.

### On PE routers

Create eBGP sessions within the dedicated VRFs with each connected CE. 

You will also need to set the Route Target (RT) and Route Distinguisher (RD) accordingly. The [L3VPN VRF](https://docs.frrouting.org/en/latest/bgp.html#l3vpn-vrfs) section of the documentation might help you to do this. You are free to choose the RT and RD for each client/site, but we ask of you that it results in a full-mesh connectivity between the sites of a same client (i.e.: don't do a hub-and-spoke connectivity).

Once you have set the corresponding RT/RD of routes, you will also need to instruct BGP to attach a segment to each route, which will be used when a packet is forwarded to do a VRF lookup. Sections [L3VPN SRv6](https://docs.frrouting.org/en/latest/bgp.html#l3vpn-srv6) and [L3VPN SRv6 SID Reachability](https://docs.frrouting.org/en/latest/bgp.html#l3vpn-srv6-sid-reachability) will describe to you how to configure this.

Finally, BGP refuses to export a route if no policy is defined. To make the project simpler for you, you can disable BGP policies using `no bgp ebgp-requires-policy`. Note however that in real deployements, you would typically define import/export policies.

Check that prefixes are indeed announced and that the RT/RD are well set using the commands provided in the section "[Useful commands to inspect BGP/SRv6/L3VPN](#useful-commands-to-inspect-bgpsrv6l3vpn)"

## Export routes

The final step is to export the routes learned within each VRF in the global BGP database, to ensure that these routes will be propagated to iBGP peers. You will then import these routes within the global BGP process, to ensure that these routes are propagated to iBGP peers. How to do this is described in the [VRF route leaking](https://docs.frrouting.org/en/latest/bgp.html#vrf-route-leaking) section.

## Test your configuration
Useful commands to inspect BGP/SRv6/L3VPN
Whenever your whole network has been configured, show that your configuration is correct by using pings. In particular, show that from a site of client (say A), you can ping another site of A, but can't ping any site of another client (say B). Once again, don't forget to specify the source address when perfoming pings.

## Useful commands to inspect BGP/SRv6/L3VPN

```bash
# BGP
show bgp summary                # show neighbors
show bgp ipv6                   # show RIB
show bgp vrf $VRF summary       # show neighbors in $VRF
show bgp ipv6 $VRF summary      # show RIB in $VRF
show bgp ipv6 vpn $VRF summary  # show prefixes with RD in $VRF
show bgp ipv6 vpn detail        # show detailed information about prefixes, such as RT, origin vrf, ...

# SRv6
show segment-routing srv6 locator
show segment-routing srv6 sid
```

## Advice

Some examples are provided in the documentation. They can be an useful help to discover how to configure BGP/SRv6/L3VPN.

Try also to work incrementaly, i.e.: configure the different steps and validate that your configuration seems correct after each step. 

## Bonus points

To go further, you can try realize any of these tasks:

- Implement a Hub-and-spoke. Change the RTs to realize a hub-and-spoke topology. Additionally, make sure that devices inside two hubs can only reach each other by going through the hub.
- Adding Route-Reflectors. For the moment, in your provider network you rely on a iBGP full mesh to exchange routes between PE routers. Try to insert a Route-Reflector inside your network to make it more scalable.
