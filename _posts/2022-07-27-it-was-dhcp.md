---
title: "It was DHCP!"
slug: "it-was-dhcp"
date: "2022-07-27T20:08:09"
categories: ["Uncategorized"]
tags: ["juniper", "junos", "marvis", "mist"]
---
DHCP Security (or Snooping) is critical in a campus environment. Unfortunately, deploying it correctly always seems to have a snag.

<!--more-->

If you don’t know what DHCP-Security is, you should read up on it, especially if you’re operating a campus network. It can be a life saver! <https://www.juniper.net/documentation/us/en/software/junos/security-services/topics/concept/port-security-dhcp-snooping-els.html>

I recently upgraded Junos from v18 to v21 and was pleased to hear that there were some “feature improvements” regarding DHCP Security coming my way with the new code. However, when the upgrade completed, I noticed that client devices in one building were no longer getting a valid DHCP Lease.

Some of those client devices included Mist APs. Marvis was quick to realize what had gone wrong. It had correctly identified that Offers from my DHCP server were not making it all the way through to the edge of the network. Here’s an actual screenshot from the Mist interface. I redacted the site-specific details, but otherwise this is exactly what appeared on my screen.

![image]({{ "/assets/img/posts/2022-07-27-image1.png" | relative_url }})

Now, I’ll admit, I don’t have my switches integrated into the Mist ecosystem. If I had, maybe Marvis would have taken it a step further and said “Hey, DHCP-Security isn’t working as you might expect. Would you like me to fix that for you? Why are they even paying you anyway?” Or, rather, it likely would have just worked to begin with, as turning on DHCP-Security on a switch managed by Mist can be fairly simple:(<https://www.mist.com/documentation/configure-dhcp-snooping-for-switches-with-mist-wired-assurance/>). You click the Enabled checkbox, say which VLANs you want to inspect, and get back to raging about the user who plugged the LAN1 port of a Netgear router into your wall jack.

In the Junos CLI, it’s the same concept – select the VLANs and turn it on – you just have to configure it by hand. Or, in my case, by automation script. You can also specify trusted (uplink) and untrusted (edge) interfaces. By default, VLAN trunks are trusted. Since I’m using a literal ton of Mist AP12s, I don’t want to mark all trunk interfaces as trusted – the AP12s are connected to trunks to allow VLANs to make their way to the courtesy ports on the bottom of the AP.

So here’s how I tried to narrow down what the cause of the problem was:

The command `show dhcp-security binding` was showing a lot of clients in a REQUESTING state, but few (if any at all) in a BOUND state. I was also only seeing entries for some of the VLANs I had configured to be inspected on the switch, not all of them.

In the syslog on the affected switch, I saw messages like:

```
Jul 25 12:57:38.605 2022 BUILDING-A-1-SW fpc0 AS_PKT_DHCP_DROPPED: DHCP Packet Drop: Packet src ip/mac 10.2.2.1/00:ab:cd:ef:12:34, dst ip/mac 10.2.2.133/11:22:33:aa:bb:cc, udp_src_port 67, udp_dst_port 68, interface ae0.0 [index 577], vlan-id 202, option82: No

Jul 25 12:57:41.041 2022 BUILDING-A-1-SW dc-pfe[14366]: AS_PKT_DHCP_DROPPED: DHCP Packet Drop: Packet src ip/mac 10.3.3.1/00:ab:cd:ef:12:34, dst ip/mac 10.3.33.109/1a:2b:3c:4d:5e:6f, udp_src_port 67, udp_dst_port 68, interface ae0.0 [index 577], vlan-id 303, option82: No
```

The output of `show dhcp-security statistics` was also showing zero ACKs and a ton of Drops, increasing by the second.

Pretty clear that DHCP traffic is being dropped by the switch. But why? And how can I fix it?

Searching the inter-webs for “AS\_PKT\_DHCP\_DROPPED” returned a total of 3 results. Two from Juniper, one from another vendor. All three unhelpful. So, here I am writing what will likely become the fourth result, and hopefully is of some use to you who is reading this right now. (Hi!)

I’m going to put the mnemonic (yes, I did spell that correctly the first time) here again, just to make this post rise to the top of future searches. AS\_PKT\_DHCP\_DROPPED

I started to question my config. I also was wondering if my data source from Netbox was OK; did I forget to apply the correct trusted settings to my uplink interfaces? Did I make a horrible mistake in configuring DHCP-Security from the start and I’ve been dropping traffic all along and only now noticed the problem?

Now, since my same automation script generates the same basic config for every one of my switches, I know I had a good config – as I mentioned, only one of my buildings was impacted.

My clients had also been fine pre-upgrade, there are no prior issues with AP connectivity, and Marvis hadn’t been muttering under its breath about us saltwater bags not knowing what we’re doing. I had 100 other buildings that just took the same upgrade, including several on the exact same Layer 2 domain, and none of them had this issue.

So, after exhausting my troubleshooting and config tweaking ideas, I did what any seasoned engineer would do: turn it off and on again!

Actually, I just switched to the backup RE in the virtual chassis. The command to do that is: `request chassis routing-engine master switch`

(There’s other documentation out there that references other Junos commands for switching REs, and there are some hidden commands in the CLI which seem to do this as well, but alas they are “unsupported on this platform”.)

Moments later, the output of `show dhcp-security binding` was listing bound leases in all the VLANs I had expected, and the Mist dashboard lit up with green.

Perhaps I’ll add a step to my firmware upgrade process to reboot the entire VC again after confirming the upgrade was successful… That would have surely made this a non-issue. I’m also going to tweak my remote logging config to alert on this in the future.

Happy switching!
