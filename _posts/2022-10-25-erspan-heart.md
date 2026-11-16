---
title: "ERSPAN ❤️"
slug: "erspan-❤️"
date: "2022-10-25T20:05:04"
categories: ["Uncategorized"]
tags: ["erspan", "gre", "junos", "linux", "ntop", "wireshark"]
---
ERSPAN is a great way to remotely troubleshoot a wired connection remotely. Even before the work-from-home era, I dreaded having to head out to a location, setup a mirror port, and connect a laptop to grab a few packets. The process of getting to the location often took longer than solving the actual issue…

<!--more-->

Encapsulated Remote Switch Port Analyzer is where a switch or router (or Linux server…) sets up a port mirror, but then instead of dumping the mirrored traffic on another physical interface, it encapsulates it inside a GRE tunnel.

Generic Routing Encapsulation allows Layer-2 data to be routed over a Layer-3 network. What this means is that you can send this mirrored traffic to any device with Layer-3 reachability to the source of the traffic.

RSPAN (without the “E”) also exists, but that requires you to be Layer-2 adjacent to a switch. RSPAN dumps the traffic on a special VLAN. Devices connected to that VLAN can then receive the traffic. ERSPAN removes that requirement. You can even send ERSPAN traffic over a VPN connection and monitor traffic on your network from home!

Actually, sending ERSPAN over a VPN isn’t a bad idea, even if you’re local to the device you’re capturing traffic from. GRE is unencrypted – anyone who is able to snoop the GRE traffic will also be able to snoop the mirrored traffic contained within.

To quickly get started with ERSPAN on a Juniper switch, use the following (ELS) config:

```
set forwarding-options analyzer sessionname input ingress interface ge-0/0/0.0
set forwarding-options analyzer sessionname input egress interface ge-0/0/0.0
set forwarding-options analyzer sessionname output ip-address 10.1.1.1
```

Then, on your workstation, fire up Wireshark and set the following custom capture filter (not display filter).

```
ip proto 0x2f
```

To set a capture filter, before starting a capture, click the Gear icon…

![image]({{ "/assets/img/posts/2022-10-25-image1.png" | relative_url }})

Then, select the interface you wish to capture on and enter the filter in the textbox at the bottom of the dialog window.

![image]({{ "/assets/img/posts/2022-10-25-image2.png" | relative_url }})

0x2F == 47 decimal. IP Protocol 47 is GRE. Yes, [IP Protocol](https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml). GRE does not use TCP or UDP! Keep this in mind if you have to adjust firewall rules along the path. (GUE also exists, which is GRE within UDP, but that’s not what is used here for ERSPAN.)

Start the capture in Wireshark, and assuming you’ve configured your source correctly and that there’s Layer-3 reachability, you should start receiving frames!

![image]({{ "/assets/img/posts/2022-10-25-image3.png" | relative_url }})

Wireshark 4 has additional display filters that make parsing this sort of traffic a bit easier.

> A syntax to match a specific layer in the protocol stack has been added. For example in an IP-over-IP packet “ip.addr#1 == 1.1.1.1” matches the outer layer addresses and “ip.addr#2 == 1.1.1.2” matches the inner layer addresses.
>
> <https://www.wireshark.org/docs/relnotes/wireshark-4.0.0.html>

Do keep in mind that ERSPAN is full-frame capture. It is not the stripped down metadata that NetFlow/sFlow sends. Duplicating traffic can overwhelm your switch/router’s uplink connection, as well as any connection on the path between the source and destinations. tl;dr – Don’t ERSPAN a 100G interface and send it to your laptop on Wi-Fi. You’ve been warned.

Many readers may wish to stop here. For troubleshooting where you need to do a simple remote PCAP to your workstation, you now know everything you need to make use of ERSPAN. To setup ERSPAN on a somewhat more permanent basis, continue on…

To receive ERSPAN traffic on a Linux server, it can be done with a simple `tcpdump -i eth0 proto 47` , but there’s a more useful, albeit more complicated, way of doing it. This method is best if you are sending the traffic to a server running [ntop](http://ntop.org) or [Zeek](http://zeek.org) for example.

To do this, you’ll need to create an ERSPAN interface, which exists in Kernel version 4.14 and later (4.16 for IPv6), which will then handle the decapsulation of the spanned traffic and dump it out the virtual interface as if you were physically connected to a mirrored port.

```
ip link add erspan1 type erspan local 10.1.1.1 remote 10.2.2.2 erspan_ver 0 
tc qdisc add dev erspan1 handle 1: ingress
tc filter add dev erspan1 parent 1: matchall skip_hw action mirred egress mirror dev erspan1
tc filter add dev erspan1 parent 1: protocol all prio 1 u32 match u32 0 0 flowid 1:1 action drop
ip link set erspan1 mtu 9000
ip link set erspan1 up
```

The local (10.1.1.1) address is the IP of your Linux server, and the remote (10.2.2.2) address is the address of the device sending the ERSPAN traffic. This means that you’ll need to create an ERSPAN interface for every ERSPAN source.

You’ll want to make sure you set the MTU of the physical interface on the Linux machine as well as on this virtual interface. (Something like `ip link set eth0 mtu 9000` should do the trick.) ERSPAN adds some overhead, so if you’re capturing full 1500-byte frames, you may exceed the normal MTU of your network.

Since this is now a proper interface in the Linux OS, these commands will also prevent the OS from being able to send traffic out this interface (such as ARP and IPv6 ND).

To view the settings of the new interface and give a quick test that it’s receiving traffic, you can run:

```
ip -d -s -s link show dev erspan1
tcpdump -i erspan1
```

The Rx counters should be increasing, and tcpdump should show the mirrored traffic without the GRE encapsulation. The Tx counters should not be increasing. The state of the interface, according to the `ip link show` command, will be UNKNOWN. This is normal.

If you want to have ntopng process the spanned traffic, it’s as simple as adding `-i` *`erspan1`* to the ntopng.conf file or the command line arguments of ntopng. nProbe is not needed, as this is not NetFlow or sFlow metadata. The same `-i erspan1` option should also work for Zeek (full disclosure, I haven’t tested it yet).

I’ll also note that Junos uses ERSPAN Type 1 (aka Version 0). Several guides to setting up ERSPAN interfaces make reference to settings that simply don’t apply to Type 1 traffic (sequence and key, for example). Other vendors may use Type 2 or Type 3 ERSPAN traffic and require additional configuration.

William Tu and Greg Rose from VMWare have written the authoritative paper on this (<http://vger.kernel.org/lpc_net2018_talks/erspan-linux.pdf>) to go along with the [commit to the kernel](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=84e54fe0a5eaed696dee4019c396f8396f5a908b) that added the feature. Hangbin Liu from RedHat has also put together a useful guide (<https://developers.redhat.com/blog/2019/05/17/an-introduction-to-linux-virtual-interfaces-tunnels>). Oracle also has documentation which provided some useful information (<https://docs.oracle.com/cd/E91266_01/RUIAB/GUID-A3B72BD1-CD24-4169-8AA7-DC72E90F1E8D.htm>).

If you want a rabbit hole to go down, read up on the `tc` command…
