---
layout: post
title: On-prem remote coding agents
categories: [general, devops, homelab, infra, ai]
tags: []
description: Deploying remote code agents
---

## Remote Code Agents

I recently tried Google's pro agent coding and really enjoyed using [jules](https://jules.google.com/) Google's remote coding agent. Claude has a similar offering called claude code cloud but I haven't been able to get the network allow list to work correctly. 

Users send a prompt to jules and choose a git repo then jules starts working on the task. It has different models like plan, ask for help, and do your own thing but it will default to full non-interactive if you don't respond to it. Jules spins up a VM, clones the repo, jules gets to work, and finishes by publishing a PR to the repository.

I really enjoyed this experience especially after CCDC events when I have a huge list of issues to solve I can copy paste them into jules. Even if they don't all get solved jules takes care of the low hanging and well documented issues.

The only problem with jules is that I can't connect it to my private gitea (not easily / safely at least) to solve this I started looking into building my own remote coding agent in the lab.

I'm hoping this agent will allow me to work on projects in my private gitea but also to experiment with different models and try building an agent. It should also enable me to connect agents to other services and find a way to safely share secrets to them.

## Benefit of remote coding agents over local

Why even run agent remotely claude code, antigravity, codex, opencode all exist and work why move to remote?

Agentic coding locally works, and lots of folks are doing it but I found especially when working through 40 tickets it's too challenging to context switch in a local dev environment. Having 40 VScode windows open with a different folder for each just doesn't work. Add a devcontainer for each and my laptop would crash.

In addition in many ways remote dev is safer from supply chain and prompt injection attacks:
- The workspace has access to limited resources defined at build
- Workspaces deleted once the task has been resolved so any persistent implant is removed

The last point has more often than not been self inflicted as I work on Realm and accidentally persist on my dev machine 😅

One thing I'd like to see from remote dev agents in the future is allowing repo maintainers to manually approve CI runs from agent pull requests.
Since most of these agents run under the github user's account their PRs are automatically approved for execution whereas an new / untrusted contributor github allows you to require maintainer approval.


## Building the thing

To get started I used [coder's AI tasks](https://coder.com/docs/ai-coder/tasks). This extends my existing coder infrastructure to support remote dev agents.

At first I tried using opencode but ran into issues with the opencode UI requiring a subdomain (turns out most of the web based IDEs aren't designed to run behind a reverse proxy openhands, opencode, bolt.diy). When that didn't work I tried my hand at building [my own agent using the pi framework](https://github.com/badlogic/pi-mono)

I called my custom agent Artificer - keeping on theme 🙂
<figure class="aligncenter">
    <img src="/assets/png/artificer-dev.png" />
</figure>

This worked pretty well and I tried working within the ethos of "agents building agents" as I worked to have pi build out the features I needed.
It worked pretty well adding:
- A TUI that conforms to the coder agentapi
- A GUI that shows git diff as the agent works
- gitea CI/CD to publish the npm package

While I really enjoyed artificer I'm still a sucker for the Anthropic models so I needed to get claude code working as well.
Claude's TUI rendering worked out of the box in coder but some features like logging with OAUTH using vault were a lil hacky.

Coder wants everything defined through terraform (reasonable) even more reasonable they have a feature called "External auth" that allows you to login to multiple remote services and have coder manage that credential and expose it through Terraform but it's for premium only.

I don't know how much coder premium costs and I'm afraid to ask so for now I'm stuck with the open-source version.

In order to work around this I'm using vault's KV and storing tokens there. The workflow is as follows:
1. Users login to vault
2. Users store Claude auth tokens in vault
3. User logs into vault through coder
<figure class="aligncenter">
    <img src="/assets/png/vault-external-auth-login.png" />
</figure>
4. User creates a workspace
5. Workspace uses vault token to authenticate to vault
6. Workspace pulls claude token stored in vault

This worked and pushed me to finish setting up policies for vault KVs.
With the user's claude token in hand I added it to the coder systemd service so it could propagate down to claude!

But wait claude still prompted me to login.
Because I was using coder's claude module to install and setup claude it was overwriting the ANTHROPIC_AUTH_TOKEN env var I set in systemd with the env var the coder agent set which defaulted to "".

systemd -> coder agent -> claude

To get around this I ended up shimming the claude executable just after it installed.

```terraform
module "claude-code" {
  count    = data.coder_task.me.enabled ? data.coder_workspace.me.start_count : 0
  source    = "registry.coder.com/coder/claude-code/coder"
  version   = "4.9.1"
  agent_id   = coder_agent.dev[0].id
  workdir   = "${local.home_dir}/${local.selected_repo.path}"
  ai_prompt = <<-EOT
  Write code good - make no mistakes:
  ${data.coder_task.me.prompt}
EOT
  dangerously_skip_permissions = true
  pre_install_script = <<-EOT
  cat << EOF > ~/.claude.json
  {
      "hasAcknowledgedDangerousSkipPermissions": true,
      "hasCompletedOnboarding": true,
      "projects": {
          "${local.home_dir}${local.selected_repo.path}": {
            "hasTrustDialogAccepted": true
          }
      }
  }
  EOF
  EOT

  post_install_script = <<-EOT
  CLAUDE_PATH=${local.home_dir}/.local/bin/claude
  
  if [ ! -f "$(dirname $CLAUDE_PATH)/notclaude" ]; then
    mv $CLAUDE_PATH "$(dirname $CLAUDE_PATH)/notclaude"
    cat << EOF > $CLAUDE_PATH
  #!/bin/bash
  . /etc/codersecrets.env
  export CLAUDE_CODE_OAUTH_TOKEN=\$CLAUDE_CODE_OAUTH_TOKEN
  exec $(dirname $CLAUDE_PATH)/notclaude "\$@"
  EOF
    chmod +x $CLAUDE_PATH
  fi

  cat << EOF > ${local.home_dir}/.claude/settings.json
  {
    "autoUpdatesChannel": "latest",
    "skipDangerousModePermissionPrompt": true
  }
  EOF

  echo "Successfully shimmed claude ✅"
  EOT
  install_claude_code = true
}
```

By re-loading the CLAUDE_CODE_OAUTH_TOKEN env var and re-exporting it just before calling claude I was able to make sure it received the correct token.

Finally claude starts up correctly and I even got a rare buddy.
<figure class="aligncenter">
    <img src="/assets/png/rare-claude-buddy.png" />
</figure>


## Conclusion

I plan to keep experimenting with artificer and looking to improve handling auth tokens.