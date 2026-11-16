---
title: "Protocol Translator"
slug: "protocol-translator"
date: "2024-10-30T12:00:00"
categories: ["Uncategorized"]
tags: ["qlab"]
---
Here’s a quick one. Sometimes you need to get two apps to talk to each other, but one only supports TCP and the other only supports UDP. (In my case, I’m hooking up my [Axis Video Doorbell](https://www.axis.com/products/axis-a8105-e/support) to trigger sound effects (and other things) in [QLab](https://qlab.app/) for some Halloween fun.)

<!--more-->

The Axis doorbell allows you to send raw TCP messages when the button is pressed. It does not support sending raw UDP. QLab can listen for raw UDP, but it only listens for a specific and more complicated protocol ([OSC](https://ccrma.stanford.edu/groups/osc/index.html)) via TCP. I didn’t want to deal with implementing the OSC checksums and datatypes by hand, so I figured I’d just convert the TCP to UDP with this neat one-liner:

```
nc -l -4 -k 53535 | nc -u -4 192.168.1.101 53535
```

This runs in a terminal on the machine that’s also running QLab. It opens a TCP listener on port 53535/tcp and then pipes it out as UDP messages to port 53535/udp, destined for the IP Address of the machine itself. QLab doesn’t listen on the loopback interface, nor does it listen on IPv6 addresses. That’s why I’ve forced IPv4 and am specifying an address that’s not 127.0.0.1. Also, yes, the ports are the same number – TCP and UDP ports are different things and can overlap. If you need to run this for other applications, the port numbers can be whatever you need them to be. You can also run this “translator” on any other machine and use it as a proxy (for example, using a machine that’s sitting on 2 otherwise separate networks).
