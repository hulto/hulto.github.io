---
layout: post
title: Securing coder with Cloudflare access
categories: [general, devops, homelab, infra]
tags: []
description: Setup and configure coder behind cloudflare tunnels with cloudflare access IDP for additional isolation.
---

## Why

After CCDC I was burned out and needed to touch grass which means a reduced patching schedule for the lab. I wanted something that could block connections before they even reach my coder server so unpatched bugs don't immediately turn into a problem.

Cloudflare Access is a good fit here - requests get authenticated at Cloudflare's edge and only authorized users ever hit the origin.

## The problem

Coder isn't just a single web UI. It has a bunch of interconnected APIs that all need to work for the platform to function:

- RPC for streaming
- `/bin/` API for serving agent binaries
- gitssh for git operations
- DERP for network routing between agents and users

Putting all of this behind a single auth-gated app breaks agents since they can't complete the OIDC flow on their own.

## The solution

I ended up splitting coder across three Cloudflare applications, each with its own access policy.

### `coder.example.com` - what users interact with

<figure class="aligncenter">
    <img src="/assets/png/coder-cf-app-config.png" />
</figure>

Straight forward to setup - users authenticate through the IDP.

### `coderint.example.com` - what agents interact with

<figure class="aligncenter">
    <img src="/assets/png/coderint-cf-app-config.png" />
</figure>

This got complicated since coder doesn't easily allow overriding the `CODER_AGENT_URL` on install. To get around this I added a custom install script that sets `CODER_AGENT_URL` to the internal domain.

I then added rules to allow agents from the home IP to connect into the service without auth.

### `coder.example.com/derp` - what both agents and users need

Originally I added DERP to the internal app but I needed users to be able to connect as well in order to build the DERP map.

While Cloudflare tunnels doesn't support `Upgrade: DERP`, I still needed to allow this endpoint for graceful failover when the direct path doesn't work.

## Next

I'm really enjoying coder and hope this workflow becomes more straight forward in the future - I started a discussion about it [here](https://github.com/coder/coder/discussions/24913).