---
layout: post
title: Homelab rebuild
categories: [general, devops, homelab, infra]
tags: []
description: Rebuilding the homelab with GCP and Harvester
---

## My homelab

I've had my homelab for a few years now and use it to test ideas, build side projects, and stay up to date on new technologies. However it's started to show it's age my servers were so old new software would fail with "intel x64 v4 instructions not supported". I figured it's time to upgrade I got rid of the old HP proliant servers and bought two Dell R730's with a combine 512GB of memory and 96 cores.

Since I was rebuilding from scratch I figured I'd try a new hypervisor since openstack had been a huge headache.

<figure class="aligncenter">
    <img src="/assets/png/harvester-cluster-capacity.png" />
</figure>

I also wanted to move a few critical services to more reliable hosting so DNS, gitea, and vault all moved to GCP.

### Capabilities

#### Support multiple users

How do we manage user access without running Windows and Active Directory?
OIDC!

This is something i've been trying to do for a while and it seems we're finally at a point where most things (even ESXi) support OIDC.

I looked at a few options in this including: Authentik, Authelia, and Okta but ended up choosing Google Identity because it's free (up to 50 users) and managed so I don't have to think about it.

Google Identity allows users to be managed through google groups (which feels weird) but works.

The downside to google identity is that you need to grant high level privileges to the IAC in order to manage it and you can only have one consent screen per project making multiple OIDC flows hard.

How do we solve this! Hashicorp vault. We're going to need a KMS anyways for managing terraform secrets might as well use vault for OIDC too. Vault also gives us a nice middleware layer to configure authentication.


#### Quickly rebuild the lab

I like to try new things including things that are core to a homelab like hypervisors.
For that reason I want to be able to pull out part of the lab and replace in quickly. The best way i've found to do this is for each part of the lab to be managed via Infrastructure as code.

I've been a Terraform user for a while now but have grown increasingly furstrated by some of the lifecycle management behavior.
I wanted to try something new so I built the homelab using [Pulumi](https://www.pulumi.com/) which has an SDK wrapper for terraform in most popular langugase including Python, Golang, and Typescript.
I tried the GoLang SDK initially but found the strict types too verbose and switched to Python.

#### Source control

Github's been a great place for me and I continue to use it for many things but with accounts and projects getting banned for offensive security work I wanted to make sure I had a safe place to work on projects. I deployed gitea as one of my two critical services to GCP. So far it's been great running as an f1-micro sometimes a medium when lots of CI/CD jobs are running.

#### Automation

Github Actions is an awesome CI/CD platform but it doesn't give me the amount of control I'd like for malware development. Specifically i'd like to prevent telemetry being sent to AV vendors.
Github Actions also gets expensive quickly if running on private repos.

I wanted a solution that gave me the control I need, was self-hosted, but still gave me the benefits for GithubActions like:
- Updated runner images
- VM isolation
- Ephemeral workloads
- Runner action ecosystem (if possible)
- Support for MacOS, Windows, and Linux

Initally I looked at github's k8s solution Actions Runner Controller (ARC) but it didn't support VM's and Windows / MacOS seemed hard to setup.
After some searching I found [GARM](https://github.com/cloudbase/garm) a CI/CD orchestration project that lets you bring your own provider with out of the box support for common platforms like AWS, GCP, and Openstack.

I wrote a provider for garm to provision VMs on my hypervisor [harvester](https://harvesterhci.io/): https://github.com/hulto/garm-provider-harvester
It was pretty easy using the documentation and examples provided.


#### Remote development environments

I've been trying to move towards remote dev environments for most projects since the servers are much more capable and I hate closing my laptop and killing a build or trying to copy large files around. An interesting benefit of this is being able to use any thin client to interact with the lab including a chromebook or an iPad.

I found [coder](https://coder.com/) which offers on-demand remote dev environment provisoning using Terraform templates. Coder's been incredible providing remote dev environments and leaning into ephemeral compute. Each workspace has a persistent volume for code but deletes the root file system everytime the VM powers down or restarts. This has been great from a security perspective even allowing me to wipe the VMs I accidentally detonated malware on.


## Conclusion

So in total the lab is now:

GCP:
- gitea - Small GCP VM
- authentication - Google Identity + Small GCP VM for Vault
- DNS - Google Domains
- backups - GCP Disk snapshots

On-prem:
- Coder
- Garm
- Harvester (hypervisor)
