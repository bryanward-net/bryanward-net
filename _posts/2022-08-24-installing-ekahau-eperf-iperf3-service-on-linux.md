---
title: "Installing Ekahau ePerf (iPerf3) Service on Linux"
slug: "installing-ekahau-eperf-iperf3-service-on-linux"
date: "2022-08-24T11:03:17"
categories: ["Uncategorized"]
tags: []
---
Ekahau makes a modified version of iPerf3 (called ePerf) that is designed to work better with their Site Survey application. Installing on Linux (RHEL, specifically in this guide), is fairly straightforward, but is poorly documented. I aim to correct that.

<!--more-->

You can download ePerf from here: <https://sw.ekahau.com/download/eperf/>

Ekahau’s official documentation can be found here: <https://support.ekahau.com/hc/en-us/articles/115008973207-Ekahau-ePerf-Server>

ePerf requires Java 8, which is also known as OpenJDK 1.8. To install this on RHEL, it’s as simple as running `sudo yum install java`. It downloads a bunch of dependencies, but should otherwise be uneventful.

Installing ePerf goes like this. Download the ePerf application:

```
wget https://sw.ekahau.com/download/eperf/iperf3-ekahau.zip
```

Make a directory to install it to, such as /opt/eperf

```
sudo mkdir /opt/eperf
```

Extract it to that directory.

```
sudo unzip iperf3-ekahau.zip -d /opt/eperf
```

Now, we’ll create a user account (named iperf) with which to run the service, otherwise it will run as root, which is a BadIdea™.

```
sudo useradd -M -r -s /sbin/nologin iperf
```

This command says to not create a home directory (-M), create a system service account instead of a named user account (-r), and to set the shell to nologin (-s). This is typical for service accounts.

Now, let’s test and make sure that the eperf software will even run before moving on. We’ll need to make the run\_eperf\_server script executable first.

```
sudo chmod +x /opt/eperf/run_eperf_server.sh
```

Now, let’s run it!

```
/opt/eperf/run_eperf_server.sh
```

This should result in something like the following output:

```
Interfaces:
ens3 (10.100.1.123)
INFO: Starting server on port 5201
```

If you get this far, you can press Ctrl-C to exit back to your command line.

Now, there’s a bug in the included shell script. The ePerf software supports a few command line options, but the script does not properly pass them along. You’ll want to edit the run\_eperf\_server.sh file and add `$@` to the end of the line so it looks like:

```
java -cp lib/iperf.jar:lib/ekahau-commons.jar:lib/log4j-1.2.17.jar:lib/jackson-annotations-2.3.0.jar:lib/jackson-core-2.3.2.jar:lib/jackson-databind-2.3.2.jar: com.ekahau.iperf.v3.Iperf3Server $@
```

This will allow you to pass arguments to the application, such as `-p` which makes the service listen on another port. (Other arguments are documented in ePerf’s included README.txt file.) You may want to do this if you’re also planning to run the vanilla iPerf3 service on it’s standard port of 5201, or if you want to run multiple instances of the ePerf service so that multiple surveyors can do active testing simultaneously, one surveyor per port. If you do this, you’ll want to keep in mind the shared bandwidth of the server itself.

```
$ /opt/eperf/run_eperf_server.sh -p 5203  
Interfaces:  
ens3 (10.100.1.123)  
INFO: Starting server on port 5203
```

Now, the other thing you’ll probably want to do is to setup ePerf as a system service, so (a) it starts automatically on system boot, and (b) restarts automatically in the event of a crash.

To do this, you’ll need to create a systemd service file for ePerf.

```
sudo echo """
[Unit]
Description=Ekahau ePerf Server
After=network.target
Wants=network-online.target

[Service]
User=iperf
Group=iperf
Restart=always
Type=simple
ExecStart=java -cp lib/iperf.jar:lib/ekahau-commons.jar:lib/log4j-1.2.17.jar:lib/jackson-annotations-2.3.0.jar:lib/jackson-core-2.3.2.jar:lib/jackson-databind-2.3.2.jar: com.ekahau.iperf.v3.Iperf3Server -p 5202
WorkingDirectory=/opt/eperf

[Install]
WantedBy=multi-user.target 
""" > /etc/systemd/system/eperf.service
```

This is set to use port 5202 instead of the default 5201. Change the number as appropriate.

Test that the service works by running:

```
sudo systemctl start eperf.service
```

```
sudo systemctl status eperf.service
```

You should see output like the following:

```
● eperf-5202.service - Ekahau ePerf Server
   Loaded: loaded (/etc/systemd/system/eperf.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-08-24 11:54:46 EDT; 1s ago
 Main PID: 294472 (java)
    Tasks: 13 (limit: 23250)
   Memory: 26.2M
   CGroup: /system.slice/eperf.service
           └─294472 /usr/bin/java -cp lib/iperf.jar:lib/ekahau-commons.jar:lib/log4j-1.2.17.jar:lib/jackson-annotations-2.3.0.jar:lib/jackson-core-2.3.2.jar:lib/jackson-databind-2.3.2.jar: com.ekahau.iperf.v3.Iperf3Server -p 5202

Aug 24 11:54:46 hostname systemd[1]: Started Ekahau ePerf Server.
Aug 24 11:54:46 hostname java[294472]: Interfaces:
Aug 24 11:54:46 hostname java[294472]:    ens3 (10.100.1.123)
Aug 24 11:54:46 hostname java[294472]: INFO: Starting server on port 5202
```

Look for the `Active: active (running)` bit and make sure there are no errors at the end of the output.

If you have iPerf3 installed on the machine (`sudo yum install iperf3` if you need it), you can fully test ePerf’s functionality by running:

```
iperf3 -c localhost -p 5202
```

Then, to make sure the service starts automatically on boot, you’ll also need to run

```
sudo systemctl enable eperf.service
```

That’s it!
