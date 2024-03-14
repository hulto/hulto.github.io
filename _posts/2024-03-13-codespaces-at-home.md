---
layout: post
title: Codespaces at home 
categories: [general, demo, devops, homelab, infra]
tags: []
description: Developing a self hosted option for remote dev contaniers that can be accessed anywhere.
---


### Codespaces

[Codespaces](https://github.com/features/codespaces) is a great option for development especially with large build workloads or limited dev resources.

I've been using codespaces a lot especially when I need to run a Windows VM along side my linux dev environment.

Codespaces uses your projects [devcontainer](https://code.visualstudio.com/docs/devcontainers/tutorial) to deploy a development environment in the cloud host by Github. I

### But we have codespaces at home

<figure class="aligncenter">
    <img src="/assets/png/Pasted image 20240313201022.png" />
</figure>

In our recent sprint to put out realm `0.1.0` I ran out of free codespaces hours and having lots of NUCs and servers at home couldn't justify paying for cloud time.

<figure class="aligncenter">
    <img src="/assets/png/Pasted image 20240313194048.png" />
</figure>


So instead I setup a home version with an intel NUC, a cloudflare tunnel and docker contexts over SSH.

#### Pre-requisites

- Docker server you want to use
- Cloudflare account
- Domain name in cloudflare
- VSCode
	- devcontainers extension

🚨 **Security Warning** 🚨
We're going to expose an SSH service to the internet make sure to disable password auth and enforce key auth only.

#### Process

1. Generate an SSH key on your local server which we'll call the `docker nuc`
2. Configure SSH to use the key to authenticate to your docker 

```
vi ~/.ssh/config
Host docker-nuc
    HostName 10.10.0.14
    User sysadmin
    IdentityFile ~/.ssh/id_rsa
```
3. Test `ssh docker-nuc` 
4. Install [docker](https://docs.docker.com/engine/install/) on the docker nuc
5. Install docker on your host system
6. On your host system create a new docker context

```bash
docker context create docker-nuc --docker "host=ssh://docker-nuc"
docker context use docker-nuc
docker ps
ssh docker-nuc docker ps
# Both ps commands should show the same containers
```

7. Test the connection by running `docker ps`
8. At this point we're able to spin up devcontainers on the remote host but only when we're connected to our local network. We could spin up a VPN to our homelab but that can make VPNing to other environments while we're connected to the dev container hard.
9. In order to connect to our docker nuc from anywhere we need to setup a tunnel from our lab to the could ☁️
10. To do this we'll use a [Cloud Flare tunnel](https://one.dash.cloudflare.com/?to=/:account/access/tunnels)
11. Create a new tunnel and select `cloudflared`

<figure class="aligncenter">
    <img src="/assets/png/Pasted image 20240313203307.png" />
</figure>

12. Give it a name 
13. Select the Operating System your NUC is running
14. Run the installer (the left hand code block)
![[Pasted image 20240313203436.png]]
15. Select a domain from the drop down
16. I recommend setting a subdomain specific to this host
17. Set the service to SSH and specify `127.0.0.1:22` as the URL

<figure class="aligncenter">
    <img src="/assets/png/Pasted image 20240313203637.png" />
</figure>

18. Click `Save Tunnel`
19. Update ssh config  to use our tunnel
	1. Update the `Hostname` and add the `ProxyCommand`

```
vi ~/.ssh/config
Host docker-nuc
    HostName docker-nuc.example.com
    User sysadmin
    IdentityFile ~/.ssh/id_rsa
    ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
```

20. Test the connection `ssh docker-nuc`
21. If the connection is succssful you should now be able to deploy your dev container to codespaces at home 🎉 
22. Open VSCode
23. Select a project that uses a devcontainer or add a devcontainer config
24. Press Ctrl+Shift+P or CMD+Shift+P and select `Dev Container: Reopen in container`
