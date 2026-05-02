---
layout: post
title: Securing coder with Cloudflare access
categories: [general, devops, homelab, infra]
tags: []
description: Setup and configure coder behind cloudflare tunnels with cloudflare access IDP for additional isolation.
---

- Why
  - Burned out after CCDC need to touch grass means - reduced patching schedule
  - CF access blocks connections before they reach the server
- the problem
  - coder has a lot of interconnected network connections
      - RPC for streaming
      - `/bin/` API for serving binaries
      - gitssh for git cloning
      - DERP for network routing
- the solution
  - Three applications
    - `coder.example.com` - What users interact with
      - ![coder application with user and admin permissions](coder-cf-app-config.png)
      - Straight forward to setup.
    - `coderint.example.com` - What agents interact with
      - ![coder internal application with bypass for home IP address](coderint-cf-app-config.png)
      - This got complicated since coder doesn't easily allow overriding the `CODER_AGENT_URL` on install
        - Added custom install script with `CODER_AGENT_URL` set to internal domain
      - Added rules to allow agents from the home IP to connect into the service without auth.
    - `coder.example.com/derp` - What both agents and users need to interact with to build mappings
      - Originally added to the internal app I needed users to be able to connect as well in order to bulid the DERP map
      - While Cloudflare tunnels doesn't support `Upgrade: DERP` still needed to allow this endpoint for graceful failover.
- Next
  - Really enjoy coder and hope this workflow becomes more straight forward in the future https://github.com/coder/coder/discussions/24913
  - Trying to put more of the lab behind CF access