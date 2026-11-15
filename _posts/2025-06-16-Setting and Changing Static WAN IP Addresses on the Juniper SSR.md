---
title: "Setting and Changing Static WAN IP Addresses on the Juniper SSR"
slug: "setting-and-changing-static-wan-ip-addresses-on-the-juniper-ssr"
date: "2025-06-16T11:33:56"
categories: ["Uncategorized"]
tags: ["juniper", "mist", "ssr"]
---
Once an SSR appliance is connected to the Mist cloud, it's easy to make changes to the config. However, if your ISP has assigned a Static IP Address, you'll need to know how to set that on the SSR using the console port before it will be able to phone-home to the Mist cloud.

Getting into the console of the SSR is similar to Junos devices. Connect to the serial port using 115200 baud 8N1. Then, log in using the `root` username and the default out-of-box password of `128tRoutes`. This is case-sensitive. The password will be changed by Mist once it phones home. You'll then be at the root shell. To get to the CLI, you now need to run `su admin` instead of `cli` like you would on Junos. This will bring you to the PCLI interface, which functions similarly to the Junos CLI, but do note that this is not Junos and the command/config language is quite different.

```
sn1234567890 login: root
Password: 
Last failed login: Mon Jun 10 12:34:56 UTC 2025 on ttyS0
There was 1 failed login attempt since the last successful login.
Last login: Mon Jun  1 13:23:45 on pts/0
+---------------------------------------+
|                                       |
|    Welcome to:                        |
|                                       |
|     | .   . ,---. . ,---. ,---. ,--.  |
|     | |   | |   | | |---' |---' |     |
|     | `---' '   ' ' '     `---' '     |
|  ---'                                 |
|        __ ___       __   __       __  |
|  |\ | |_   |  |  | /  \ |__) |_/ (_   |
|  | \| |__  |  |/\| \__/ | \  | \ __)  |
|                                       |
| Session Smart Networking Platform ... |
+---------------------------------------+
[root@sn1234567890 ~]# 
[root@sn1234567890 ~]# cli
-bash: cli: command not found
[root@sn1234567890 ~]# su admin
Starting the PCLI...
admin@node0.900102abcdef# 
```

If you wait too long at this prompt, you might get kicked back to the root shell. If this happens, just run `su admin` again.

```
% Error: Session timed out after 900 seconds
Exiting the PCLI
[root@sn1234567890 ~]# su admin
Starting the PCLI…
admin@node0.900102abcdef# 
```

Your prompt may say `admin@node.router` instead of `node0` and the hostname, MAC Address, or Serial Number of the device. Adjust your commands accordingly to match the environment of your SSR. If your prompt says node.router, you'll want to use commands like `configure authority router router node node`. If your prompt has node0 and a MAC Address, you'll want to use commands like `configure authority router 900102abcdef` `node node0`. I provide example using both throughout this article.

Once at this prompt, setting a static IP Address is pretty straightforward. However, if your SSR comes out of the box with older code (such as version 5.4.4), the [documentation provided by Juniper](https://www.juniper.net/documentation/us/en/software/mist/mist-wan/topics/topic-map/ssr-onboarding-with-static-ip.html) is not a simple copy and paste. You'll know if you're in this situation if you see the following messages when you try to apply the config the documentation tells you to use:

```
admin@node.router# configure authority router router node node device-interface
ge-0-0 network-interface ge-0-0-intf dhcp disabled
*admin@node.router# configure authority router router node node device-interface
 ge-0-0 network-interface ge-0-0-intf address 192.0.2.101 prefix-length 24
*admin@node.router# configure authority router router node node device-interface
 ge-0-0 network-interface ge-0-0-intf address 192.0.2.101 gateway 192.0.2.1

*admin@node.router# commit force
Validating, then committing…
% Error: Failed to commit:
1. PCI address, VMBus UUID or Interface Name is required for device-interface
ge-0-0 when type is ethernet.
```

The issue here is that the interfaces names may be different in the version of firmware your device is runnig that the version for which the documentation was written. You can check what version firmware the SSR is running by using the following command:

```
admin@node.router#> show system version
Tue 2025-06-10 14:34:56 UTC

======== ====== ========= ====================== ==================
Router   Node   Version   Build Date             Package
======== ====== ========= ====================== ==================
router   node   5.4.4     2022-02-17T17:52:07Z   128T-5.4.4-9.el7

Completed in 0.05 seconds
```

```
admin@node0.900102abcdef# show system version
Mon 2025-06-10 15:45:07 UTC
Retrieving system version...

============== ======= ========= ======== ====================== ===============
 Router         Node    Version   Status   Build Date             Package
============== ======= ========= ======== ====================== ===============
 900102abcdef   node0   6.3.4     r2       2025-02-05T03:12:54Z   128T-6.3.4-7.
                                                                  r2.el7

Completed in 0.05 seconds
```

**The solution is simple. Just remove (or add) the `-intf` suffix from the name of the network-interface.**

You can see if an interface is configured for DHCP or a Static IP Address with the command `show config run flat | grep -P "**ge-0-0** (dhcp|address)"`. Replace ge-0-0 with the name of the interface you want to check. If you see a single line that mentions dhcp, then the interface is configured to request a DHCP lease. If you see lines mentioning ip-address, prefix-length, and gateway config, then it's set for the defined Static IP address.

```
admin@node0.900102abcdef# show config run flat | grep -P "ge-0-0 (dhcp|address)"
 
Piping output...
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 address 192.0.2.101 ip-address     192.0.2.101
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 address 192.0.2.101 prefix-length  24
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 address 192.0.2.101 gateway        192.0.2.1


admin@node0.900102abcdef# show config run flat | grep -P "ge-0-1 (dhcp|address)"
 
Piping output...
config authority router 900102abcdef node node0 device-interface ge-0-1 network-interface ge-0-1 dhcp                   v4
admin@node0.900102abcdef#
```

To change the address assignment, you must first remove the existing Static IP config like so:

```
admin@node0.900102abcdef# configure authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 delete address 192.0.2.101
% Warning: You are about to make configuration changes to a router that is
managed by the Mist Cloud. These changes could be overwritten at any time by the
Mist Cloud.
Are you sure you want to delete item '[ip-address=192.0.2.101]' [y/N]: y
*admin@node0.900102abcdef#

*admin@node0.900102abcdef# show config candidate flat | grep "ge-0-0 address"
Piping output...
Pipe failed: Command 'grep' exited with non-zero exit status 1
```

(The error from 'grep' indicates that no config was found.)

And then assign a different Static IP Address:

```
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 dhcp disabled
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 address 192.0.2.111 prefix-length  25
config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 address 192.0.2.111 gateway        192.0.2.1
```

Or enable DHCP:

```
*admin@node0.900102abcdef# config authority router 900102abcdef node node0 device-interface ge-0-0 network-interface ge-0-0 dhcp

# Nothing to show

*admin@node0.900102abcdef#
```

To "rollback" the candidate config to the version that is currently running, use:

```
*admin@node0.900102abcdef# restore config running
% Warning: You are about to make configuration changes to a router that is
managed by the Mist Cloud. These changes could be overwritten at any time by the
Mist Cloud.
Are you sure you want to discard uncommitted changes from the candidate config f
or user admin? [y/N]: y
Discarding uncommitted changes...
Candidate configuration changes successfully discarded for admin
*admin@node0.900102abcdef# 
```

The SSR equivalent of `show | compare` is `compare config`. It will display the difference between the running and the candidate in full stanzas, not in flat one-liner syntax.

Once you're satisfied with your config changes, run `commit`. This checks the config for any warnings/errors before prompting you to apply it.
