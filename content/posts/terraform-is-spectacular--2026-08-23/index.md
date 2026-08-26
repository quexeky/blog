---
title: Terraform is Spectacular
date: 2026-08-23T11:06:00+10:00
description: Terraform is an absolute gift to humanity and everyone should use it
cover:
  relative: true
  image: hashicorp-terraform_ondark.svg
  alt: Hashicorp and Terraform Logo
showToc: true
---
Until yesterday, the rough extent of my experience with Infrastructure as Code (IaC), if it can be called that, is Docker compose files. They're nice enough, but personally, I find them ugly and a pain in the ass to maintain. Terraform though? Oh man. Maybe it's just the tooling around it, but there's something that's so nice about the fact that I can just write a HCL file (I think that's what it's called?), build it in like 10 seconds to validate everything, and then just have a deterministically built container in like another 30 seconds after that.

I'm in the process of setting up some Coder templates for Recadia, which is why I've been putting in this effort to build it, but so far I haven't really run in to many major difficulties. The most annoying thing so far has been trying to parallelise some setup / installation scripts, but even then it's really not that bad. The environment that I've set up so far is like so:

```hcl
terraform {
  required_providers {
    coder = {
      source = "coder/coder"
    }
    docker = {
      source = "kreuzwerker/docker"
    }
    external = {
      source = "hashicorp/external"
    }
  }
}

variable "docker_socket" {
  default     = ""
  description = "(Optional) Docker socket URI"
  type        = string
}

provider "docker" {
  host = var.docker_socket != "" ? var.docker_socket : null
}

data "coder_provisioner" "me" {}
data "coder_workspace" "me" {}
data "coder_workspace_owner" "me" {}

# --- Parameters ---

data "coder_parameter" "gpg_key" {
  name         = "gpg_key"
  display_name = "GPG Private Key (Optional)"
  description  = "Paste your ASCII-armored GPG private key to automatically sign Git commits."
  type         = "string"
  default      = ""
  mutable      = true
}

data "coder_parameter" "branch" {
  name         = "branch"
  display_name = "Git branch to automatically use (Optional)"
  description  = "Paste the name of a new or existing git branch to use automatically"
  type         = "string"
  default      = ""
  mutable      = true
}

# --- Locals ---

locals {
  username  = data.coder_workspace_owner.me.name
}

# --- Agent ---

resource "coder_agent" "main" {
  arch           = data.coder_provisioner.me.arch
  os             = "linux"
  startup_script = <<-EOT
    set -e
    if [ ! -f ~/.init_done ]; then
      cp -rT /etc/skel ~
      touch ~/.init_done
    fi

    # Configure GPG Commit Signing if a key was provided
    if [ -n "$GPG_KEY" ]; then
      echo "Importing GPG signing key..."
      echo "$GPG_KEY" | gpg --batch --import --yes || true
      
      # Extract key ID and configure git
      KEY_ID=$(gpg --list-secret-keys --keyid-format=long | grep -E '^sec' | awk '{print $2}' | cut -d'/' -f2 | head -n 1)
      if [ -n "$KEY_ID" ]; then
        git config --global user.signingkey "$KEY_ID"
        git config --global commit.gpgsign true
        echo "Git configured to sign commits with key: $KEY_ID"
      fi
    fi

  EOT

  env = {
    GIT_AUTHOR_NAME     = coalesce(data.coder_workspace_owner.me.full_name, data.coder_workspace_owner.me.name)
    GIT_AUTHOR_EMAIL    = "${data.coder_workspace_owner.me.email}"
    GIT_COMMITTER_NAME  = coalesce(data.coder_workspace_owner.me.full_name, data.coder_workspace_owner.me.name)
    GIT_COMMITTER_EMAIL = "${data.coder_workspace_owner.me.email}"
    GPG_KEY             = data.coder_parameter.gpg_key.value
    GIT_BRANCH          = data.coder_parameter.branch.value
    LANG                = "en_US.UTF-8"
    LANGUAGE            = "en_US:en"
    LC_ALL              = "en_US.UTF-8"
  }

  metadata {
    display_name = "CPU Usage"
    key          = "0_cpu_usage"
    script       = "coder stat cpu"
    interval     = 10
    timeout      = 1
  }

  metadata {
    display_name = "RAM Usage"
    key          = "1_ram_usage"
    script       = "coder stat mem"
    interval     = 10
    timeout      = 1
  }

  metadata {
    display_name = "Home Disk"
    key          = "3_home_disk"
    script       = "coder stat disk --path $${HOME}"
    interval     = 60
    timeout      = 1
  }
}


# --- Language installation ---

resource "coder_script" "apt_setup" {
  count              = 1
  agent_id           = coder_agent.main.id
  display_name       = "Setup apt"
  icon               = "/icon/code.svg"
  run_on_start       = true
  start_blocks_login = true
  script = templatefile("${path.module}/apt-setup.sh.tftpl", {})
}

resource "coder_script" "install_rust" {
  count              = 1
  agent_id           = coder_agent.main.id
  display_name       = "Install Rust"
  icon               = "/icon/code.svg"
  run_on_start       = true
  start_blocks_login = true
  script = templatefile("${path.module}/install-languages.sh.tftpl", {})
  }

resource "coder_script" "install_node" {
  count              = 1
  agent_id           = coder_agent.main.id
  display_name       = "Install Node"
  icon               = "/icon/code.svg"
  run_on_start       = true
  start_blocks_login = true
  script = templatefile("${path.module}/install-node.sh.tftpl", {})
}

# --- IDE modules ---

module "code-server" {
  count    = data.coder_workspace.me.start_count
  source   = "registry.coder.com/coder/code-server/coder"
  version  = "~> 1.0"
  agent_id = coder_agent.main.id
  order    = 1
  extensions = [...]
  additional_args = "--disable-workspace-trust"
}

module "zed" {
  count    = data.coder_workspace.me.start_count
  source   = "registry.coder.com/coder/zed/coder"
  version  = "~> 1.0"
  agent_id = coder_agent.main.id
  folder   = "/home/coder"
  order    = 5
}

# --- Git clone ---

module "git-clone" {
  count    = data.coder_workspace.me.start_count
  source   = "registry.coder.com/coder/git-clone/coder"
  version  = "~> 2.0"
  agent_id = coder_agent.main.id
  url      = "git@git.recadia.dev:recadia/recadia.git"
  branch_name = "main"
  post_clone_script = <<-EOT
    #!/bin/bash
    if [ -n "$GIT_BRANCH" ]; then
      echo "Switching to branch $GIT_BRANCH..."

      git fetch origin --prune

      if git rev-parse --verify "origin/$GIT_BRANCH" >/dev/null 2>&1; then
        echo "Found remote branch origin/$GIT_BRANCH. Checking out..."
        git switch "$GIT_BRANCH"
      else
        echo "Remote branch not found. Creating new local branch $GIT_BRANCH..."
        git switch -c "$GIT_BRANCH"
      fi
    else
      echo "Staying on main!"
    fi
  EOT
}

# --- Docker resources ---

resource "docker_volume" "home_volume" {
  name = "coder-${data.coder_workspace.me.id}-home"
  # Protect the volume from being deleted due to changes in attributes.
  lifecycle {
    ignore_changes = all
  }
  # Add labels in Docker to keep track of orphan resources.
  labels {
    label = "coder.owner"
    value = data.coder_workspace_owner.me.name
  }
  labels {
    label = "coder.owner_id"
    value = data.coder_workspace_owner.me.id
  }
  labels {
    label = "coder.workspace_id"
    value = data.coder_workspace.me.id
  }
  # This field becomes outdated if the workspace is renamed but can
  # be useful for debugging or cleaning out dangling volumes.
  labels {
    label = "coder.workspace_name_at_creation"
    value = data.coder_workspace.me.name
  }
}

resource "docker_network" "private_network" {
  name = "network-${data.coder_workspace.me.id}"
}

resource "docker_container" "dind" {
  image      = "docker:dind"
  privileged = true
  name       = "dind-${data.coder_workspace.me.id}"
  entrypoint = ["dockerd", "-H", "tcp://0.0.0.0:2375"]
  networks_advanced {
    name = docker_network.private_network.name
  }
}

resource "docker_container" "workspace" {
count = data.coder_workspace.me.start_count
  image = "codercom/enterprise-base:ubuntu"
  name  = "coder-${data.coder_workspace_owner.me.name}-${lower(data.coder_workspace.me.name)}"

  # Hostname makes the shell more user friendly: coder@my-workspace:~$
  hostname = data.coder_workspace.me.name
  user = "coder"
  # Use the docker gateway if the access URL is 127.0.0.1
  entrypoint = ["sh", "-c", replace(coder_agent.main.init_script, "/localhost|127\\.0\\.0\\.1/", "host.docker.internal")]
  env        = [
    "CODER_AGENT_TOKEN=${coder_agent.main.token}",
    "DOCKER_HOST=${docker_container.dind.name}:2375"  
  ]
  host {
    host = "host.docker.internal"
    ip   = "host-gateway"
  }
  volumes {
    container_path = "/home/coder"
    volume_name    = docker_volume.home_volume.name
    read_only      = false
  }

  # Add labels in Docker to keep track of orphan resources.
  labels {
    label = "coder.owner"
    value = data.coder_workspace_owner.me.name
  }
  labels {
    label = "coder.owner_id"
    value = data.coder_workspace_owner.me.id
  }
  labels {
    label = "coder.workspace_id"
    value = data.coder_workspace.me.id
  }
  labels {
    label = "coder.workspace_name"
    value = data.coder_workspace.me.name
  }
  networks_advanced {
    name = docker_network.private_network.name
  }
}
```

```shell
#!/bin/bash
set -e

UNIT_NAME="install-node"

echo "Waiting for dependencies to install"
coder exp sync want "$UNIT_NAME" "install_deps"
coder exp sync start "$UNIT_NAME"


echo "Installing Node.js & pnpm..."

if command -v node >/dev/null 2>&1; then
  echo "Node.js: $(node --version)"
else
  echo "Installing Node.js 22..."
  curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
  sudo apt-get install -y -qq nodejs
  echo "Installed Node.js: $(node --version)"
fi

if command -v pnpm >/dev/null 2>&1; then
  echo "pnpm: $(pnpm --version)"
else
  echo "Installing pnpm..."
  wget -qO- https://get.pnpm.io/install.sh | ENV="$HOME/.bashrc" SHELL="$(which bash)" bash -

  source /home/coder/.bashrc
  echo "Installed pnpm: $(pnpm --version)"
fi

echo "Finished installing Node.js & pnpm!"
coder exp sync complete "$UNIT_NAME"
```

```shell
#!/bin/bash
set -e

UNIT_NAME="install-rust"

echo "Waiting for dependencies to install"
coder exp sync want "$UNIT_NAME" "install_deps"
coder exp sync start "$UNIT_NAME"

if command -v rustc >/dev/null 2>&1 || [ -f "$HOME/.cargo/bin/rustc" ]; then
  RUSTC=$${HOME}/.cargo/bin/rustc
  command -v rustc >/dev/null 2>&1 && RUSTC=rustc
  echo "Rust: $($RUSTC --version)"
else
  echo "Installing Rust..."
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
  echo "Installed Rust: $($HOME/.cargo/bin/rustc --version)"

  source ~/.bashrc
  . "$HOME/.cargo/env" 

  echo "Installing SQLx and reauditor..."

  cargo install sqlx-cli
  cargo install --git https://git.recadia.dev/recadia/reauditor.git
fi


echo "Rust setup complete."

coder exp sync complete "$UNIT_NAME"
```

```shell
#!/bin/bash
UNIT_NAME="install_deps"
coder exp sync start "$UNIT_NAME"

sudo apt-get update -qq

sudo apt-get install -y openssl ca-certificates libssl-dev pkg-config

trap 'coder exp sync complete "install_deps" > /dev/null 2>&1 || true' EXIT

coder exp sync complete "$UNIT_NAME"
```

I grabbed a few Docker-in-Docker blocks from an example that @DecDuck had set up, which just worked the first time (thank you!), and after scavenging around in the Coder docs, I found their [Startup Dependencies](https://coder.com/docs/admin/templates/startup-coordination/usage), which ended up being a massive help for parallelisation, due to the fact that the node / pnpm apt-get installation commands would conflict with updating and installing libssl-dev and such things (which makes sense after all - they're all just running on the same machine). But now I've basically just got a workspace template that will let me choose a branch to work in, and automatically install all of the tools that I need in a completely fresh environment, whenever I need it. It's far from the most efficient use of resources, but MAN is it nice.

Will definitely be using terraform (and probably Coder) again for such things.
