---
title: "Upgrading Junos the Hard Way"
slug: "upgrading-junos-the-hard-way"
date: "2020-06-08T22:49:39"
categories: ["Uncategorized"]
tags: ["juniper"]
---
There are some known issues with Juniper EX Switches and some earlier builds of Junos 18 regarding having insufficient free storage space when attempting to upgrade the Junos software. I’m going to detail 2 methods of getting the upgrade to work while in this situation.

<!--more-->

### One

This solution has worked for 90%+ of my production switches. It involves some cleanup of old files that may be residing on the storage, copying the new software installer to a temp directory, applying a patch which ~~puts the installer under local anesthesia~~ makes the installer calculate free space a bit differently, and then forces the install to clean things up as it goes.

```
show system storage all-members | no-more
request system storage cleanup all-members no-confirm | no-more
request system snapshot all-members delete snap* | no-more
request system snapshot recovery delete * all-members | no-more
show system storage all-members | no-more
```

You should have somewhere around 40% free space. I’m not sure how much is needed exactly, but things seem to work if you have that much free space. Then, proceed with installing the patches:

```
file copy ftp://USER:PASSWORD@SERVER/os-package.tgz /var/tmp/os-package.tgz
file copy ftp://USER:PASSWORD@SERVER/package-hooks-ex.tgz /var/tmp/package-hooks-ex.tgz
request system software add /var/tmp/os-package.tgz
request system software add /var/tmp/package-hooks-ex.tgz
requests system software add ftp://USER:PASSWORD@SERVER/junos-arm-32-18.2R3-S3.11.tgz force unlink
```

The `os-package` and `package-hooks-ex` files are the patches Juniper has released in order to work around the storage space issue. They can be downloaded from here: <https://kb.juniper.net/InfoCenter/index?page=content&id=KB31198&actp=METADATA> Get your junos software file from the [usual place](https://support.juniper.net/support/downloads/).

You may see a reference in various documentation to using the `no-copy` option of `request system software add`. From what I have observed, this does not work on a Virtual Chassis, as it prevents copying the software from the master to the other members of the VC. It may succeed in installing on the master, but will fail on the others, potentially leaving you with different software running on your VC (a BadIdea™)

Once the install completes, you’ll be given the usual `NOTICE: 'pending' set will be activated at next reboot…` Check that it actually worked on all members by running `show version` and looking for the line starting with `Pending:`. Reboot the switch/VC when ready.

Update: If the installer still complains about storage space at this point, do another `request system storage cleanup all-members` and try installing junos again with the force and unlink options. There is a footnote in the Juniper docs for the patches that says it might not work on the first try. If you’re installing on a single-member VC or standalone switch, then you can try adding the no-copy option as well.

When the switch/VC comes back online, you’ve got to remove those two patches as indicated by the new warning message that appears when you log into the switch:

```
At least one package installed on this device has limited support.
Run 'file show /etc/notices/unsupported.txt' for details.
```

To delete the patches, run the following, including the reboot.

```
request system software delete os-package
request system software delete package-hooks-platform
request system reboot at now
```

You might see one of them uninstall and not the other or give some error like `/usr/sbin/pkg: package os-package is not installed or not in active set.` Don’t worry about it and reboot again.

### Two

If that doesn’t work, and you’ve cleared out all the storage you can think of (old junos files in /packages/db/, unnecessary files in /var/tmp and /var/log), then this method is what did the trick for me.

Get a USB stick and format it as FAT32. Copy the Junos file you wish to install to it (not the install media one, just the normal junos tgz file).

Stick the stick in and mount it:

```
start shell
su
cd /var/tmp
mkdir usb
chmod 777 usb
mount -t msdosfs /dev/da1s1 /var/tmp/usb
mount | grep usb
exit
exit
```

You should now be back at the Junos CLI. That `mount | grep` usb line is to check and make sure Junos hasn’t decided to change the mount point on you…

```
% mount | grep usb
/dev/da1s1 on /.mount/var/tmp/usb (msdosfs, local)
```

See? It changed it here…

Anyway, now that you’re in the CLI, and you have free storage space, let’s install those patches and then install Junos from the USB stick directly:

```
file copy ftp://USER:PASSWORD@SERVER/os-package.tgz /var/tmp/os-package.tgz
file copy ftp://USER:PASSWORD@SERVER/package-hooks-ex.tgz /var/tmp/package-hooks-ex.tgz
request system software add /var/tmp/os-package.tgz
request system software add /var/tmp/package-hooks-ex.tgz
request system software add /.mount/var/tmp/usb/junos-arm-32-18.2R3-S3.11.tgz force no-copy member 0
```

Here we’re going to use no-copy, but not unlink. We’re also going to install it on just the VC member that has the stick inserted (0 in this case). One VC member cannot (easily) access a mounted USB stick of another VC member. Wait for the install to complete and see your `NOTICE: 'pending' set will be activated at next reboot` message.

Unmount the USB stick and plug it into the next VC member:

```
start shell
su
umount /dev/da1s1
exit
exit

request session member 1
start shell
su
cd /var/tmp
mkdir usb
chmod 777 usb
mount -t msdosfs /dev/da1s1 /var/tmp/usb
mount | grep usb
exit
exit
```

You should now be at the CLI of member 1 and it should have the USB stick mounted on itself. Proceed with the install, on `member 1` this time:

```
request system software add /.mount/var/tmp/usb/junos-arm-32-18.2R3-S3.11.tgz force no-copy member 1
```

You may also need to specify the `no-validate` option. I did encounter the below error on some VC members:

```
...
Virtual Chassis Control Protocol: [edit system commit]
Virtual Chassis Control Protocol:   'factory-settings'
Virtual Chassis Control Protocol:     Factory default configuration not allowed on virtual chassis
...
ERROR: Current configuration not compatible with junos-arm-32-18.2R3-S3.11.tgz
...
```

And there was no cheerful “`NOTICE: 'pending' set will be activated at next reboot`” at the end of the output. Re-running the `request system software add` command with `no-validate` specified did the trick.

Rinse and repeat for all your VC members. The output of `show version | match Pending` should show the new Junos version ready to go on all of them. Unmount the USB stick and remove it, reboot the VC, cross your fingers, and cleanup the patched files when it comes back:

```
request system software delete os-package
request system software delete package-hooks-platform
request system reboot at now
```
