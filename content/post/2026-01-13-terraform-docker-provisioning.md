---
layout: post
title: "Automating Docker Container Deployment on Proxmox with Terraform"
subtitle: "Using SSH Agent authentication to provision Docker containers in LXC containers - and why Ansible is next"
date: 2026-01-13 14:00:00
author: "Rene Welches"
image: "/img/red-hook-crit.jpg"
publishDate: 2026-01-13 14:00:00
tags:
  - Homelab
  - Proxmox
  - Terraform
  - Docker
  - Infrastructure as Code
categories: [Homelab, Proxmox]
URL: "/2026/01/13/terraform-docker-provisioning"
---

## Introduction

In my homelab journey, I've been working on automating the deployment of containerized applications on Proxmox. My current setup uses Terraform to create LXC containers and provision them with Docker containers - all automated through SSH-based provisioners.

This post walks through my Terraform configuration that deploys Docker-enabled LXC containers on Proxmox, focusing on how I use SSH agent authentication to automatically configure services like Open WebUI, SearXNG, and n8n. I'll also discuss why this approach works well for getting started but why I'm planning to migrate to Ansible for better configuration management.

## The Problem: Bridging Infrastructure and Application Deployment

When building a homelab, you typically need to:

1. Create the infrastructure (VMs or containers)
2. Install prerequisites (Docker, packages, etc.)
3. Deploy and configure applications
4. Manage updates and changes

Terraform excels at the first step - infrastructure provisioning. But what about steps 2-3? This is where things get interesting, and where the lines between infrastructure and configuration management blur.

## My Current Terraform Setup

My setup uses the [bpg/proxmox](https://registry.terraform.io/providers/bpg/proxmox/latest/docs) provider to create unprivileged LXC containers with Docker pre-installed, then uses Terraform's `remote-exec` and `file` provisioners to deploy Docker containers via SSH.

### The Foundation: Docker-Ready LXC Templates

The key to this workflow is having a custom Debian 13 template that includes:
- Docker installed and running
- SSH key pre-provisioned for root user
- Proper LXC nesting features enabled

This template eliminates the need to install Docker during provisioning, significantly speeding up deployment time.

### SSH Agent Authentication: The Magic Ingredient

The most critical part of my setup is the SSH authentication strategy. Instead of embedding passwords or SSH keys in my Terraform configuration, I leverage the SSH agent:

```hcl
provisioner "remote-exec" {
  inline = [
    "apt-get update",
    "apt-get upgrade -y",
    "docker run -d --restart unless-stopped -p 80:8080 ..."
  ]
  connection {
    type  = "ssh"
    user  = "root"
    host  = split("/", self.initialization[0].ip_config[0].ipv4[0].address)[0]
    agent = true  # Uses SSH agent for authentication
  }
}
```

The `agent = true` setting is crucial here. It tells Terraform to use the SSH agent for authentication rather than requiring passwords or key files.

### Setting Up SSH Agent

Before running Terraform, you need to ensure your SSH agent is running and has the correct key loaded:

```bash
# Start ssh-agent if not running
eval $(ssh-agent)

# Add your SSH key (the one provisioned in the template)
ssh-add ~/.ssh/id_rsa

# Verify the key is loaded
ssh-add -l
```

Good news for Mac users,  this can be done automatically and you don't have to do this with every new session.

```bash
cd ~/.ssh
sudo vim config
```

(or any other editor you prefer). And then add to the host* section: 

```bash
Host*
  UseKeychain yes
  AddKeysToAgent yes
```

`UseKeychain yes` (macOS-specific)
It tells SSH to look for your private keys in the macOS Keychain instead of ~/.ssh/id_rsa. Since you did ssh-add a key it should be stored there. SSH will then retrieve it from the Keychain instead of asking you every time for a passphrase.

`AddKeysToAgent yes` (macOS-specific)
When you connect to a host, SSH automatically adds your key to the SSH agent (ssh-agent). The agent runs in the background and holds decrypted keys in memory, so you don't need to type your passphrase repeatedly.

Voilá no more passphrase when connecting to server which accept your SSH key.

This approach provides several benefits:

- **No secrets in code**: SSH keys never appear in Terraform files
- **Secure authentication**: Keys remain encrypted by the SSH agent
- **Simple workflow**: Once the agent is configured, it "just works"
- **Multiple key support**: The agent can manage different keys for different hosts

### Dynamic IP Address Extraction

One clever aspect of the configuration is how it extracts the container's IP address for SSH connections:

```hcl
host = split("/", self.initialization[0].ip_config[0].ipv4[0].address)[0]
```

Since Proxmox stores IP addresses in CIDR notation (e.g., `192.168.1.100/24`), we use `split()` to extract just the IP address portion for SSH connectivity.

## Real-World Examples: Services I'm Running

Let me walk through a few examples from my setup:

### Open WebUI Container

This container runs [Open WebUI](https://github.com/open-webui/open-webui), a web interface for Ollama AI models, where Ollama is running on another remote machine:

```hcl
resource "proxmox_virtual_environment_container" "open-webui-container" {
  node_name = var.proxmox_node

  unprivileged = true
  features {
    nesting = true  # Required for Docker
  }

  initialization {
    hostname = "open-webui"
    user_account {
      password = var.proxmox_host_default_pwd
    }
    ip_config {
      ipv4 {
        address = "${var.static_ips.open_webui}/32"
        gateway = "192.168.1.1"
      }
    }
  }

  # Resource allocation
  cpu {
    cores = 2
  }
  memory {
    dedicated = 1536
  }
  disk {
    datastore_id = "local-lvm"
    size         = 20
  }

  # Upload environment file
  provisioner "file" {
    source      = "openwebui/docker.env"
    destination = "/tmp/docker.env"
    connection {
      type  = "ssh"
      user  = "root"
      host  = split("/", self.initialization[0].ip_config[0].ipv4[0].address)[0]
      agent = true
    }
  }

  # Deploy Docker container
  provisioner "remote-exec" {
    inline = [
      "apt-get update",
      "apt-get upgrade -y",
      "docker run -d --restart unless-stopped -p 80:8080 -e OLLAMA_BASE_URL=${var.ollama_host} --env-file /tmp/docker.env -v open-webui:/app/backend/data --name open-webui ghcr.io/open-webui/open-webui:main"
    ]
    connection {
      type  = "ssh"
      user  = "root"
      host  = split("/", self.initialization[0].ip_config[0].ipv4[0].address)[0]
      agent = true
    }
  }
}
```

This example demonstrates:

- Using the `file` provisioner to upload configuration file which contains docker environment variable
- Running system updates before deploying applications
- Deploying Docker containers with persistence (volumes)
- Using environment variables for dynamic configuration

## What Works Well

After using this setup for several deployments, here's what I appreciate:

1. **Fast deployment**: Once the template is ready, spinning up new services takes just a few minutes
2. **Infrastructure as Code**: Everything is version controlled and reproducible
3. **Secure authentication**: SSH agent keeps keys secure and out of configuration files
4. **Single command deployment**: `terraform apply` handles everything from infrastructure to application
5. **Static IP management**: Easy to manage network configuration through variables

## The Limitations: Why I'm Moving to Ansible

While this Terraform-based approach works, it has several limitations that are pushing me towards adding Ansible:

### 1. Configuration Management Isn't Infrastructure

Terraform is designed for infrastructure lifecycle management - creating, updating, and destroying resources. Using it for configuration management (installing packages, deploying apps) is working against its design philosophy.

**Problem**: If a Docker container fails or you need to update it, Terraform won't detect this drift unless the infrastructure changes. You need to manually taint resources or destroy and recreate them. In my case I often see myself deleting the LXCs and let terraform re-create them. This becomes especially annoying when you are still trying out different configurations. For example, I added a SearXNG LXC and enabled on the Open WebUI LXC the [Web Search](https://docs.openwebui.com/features/web-search/searxng) feature. With every configuration change I had to delete the Open WebUI LXC.

### 2. No Configuration Drift Detection

Terraform tracks the state of infrastructure but not the configuration inside containers:

- If someone manually stops or modifies a Docker container, Terraform will not know
- System package updates are not tracked
- Configuration file changes are not managed

### 3. Error Handling and Debugging

When a provisioner fails, debugging is harder:

- Limited output visibility
- No easy way to re-run just the failed configuration step
- Must destroy and recreate to retry

### 4. Separation of Concerns

Mixing infrastructure (LXC containers) with application deployment (Docker containers) in the same Terraform configuration makes it harder to:

- Update applications without touching infrastructure (you basically do it by hand or delete the VM/LXC)
- Manage configuration across multiple hosts
- Apply different update schedules for infrastructure vs. applications

## The Better Approach: Terraform + Ansible

My planned architecture separates responsibilities:

**Terraform handles**:

- Creating LXC containers
- Networking configuration
- Storage allocation
- Basic container initialization

**Cloud-Init handles**:

- Adding a default user (besides root)
- Maybe adding some user scripts
- Run updates

**Ansible handles**:

- Package installation and updates
- Docker container deployment
- Configuration file management
- Service orchestration
- Configuration drift detection and remediation

This separation provides:

- **True idempotency**: Ansible ensures desired state regardless of current state
- **Better error handling**: Ansible can retry failed tasks and has better debugging
- **Configuration management**: Track and manage configuration drift
- **Reusability**: Ansible playbooks can be used independently of Terraform
- **Easier updates**: Update applications without touching infrastructure


## Conclusion

My current Terraform-based Docker provisioning setup demonstrates that you *can* use Terraform for both infrastructure and configuration management. The SSH agent authentication provides a secure way to connect and configure containers without embedding secrets in code.

However, just because you *can* doesn't mean you *should*. While this approach works fine for quick deployments and initial setup, Ansible is the better tool for configuration management:

- Better idempotency
- Drift detection
- Easier updates
- Separation of concerns
- More maintainable long-term

My next steps involve:

1. Keeping Terraform for LXC container provisioning
2. Migrating Docker container deployment to Ansible
3. Creating reusable Ansible roles for common services
4. Setting up proper configuration management workflows
5. Adding Cloud-init

## Resources

- [Securing Proxmox API Tokens with Apple Keychain](/2025/12/09/proxmox-terraform-keychain/)
- [bpg/proxmox Terraform Provider](https://registry.terraform.io/providers/bpg/proxmox/latest/docs)
- [Terraform Provisioners Documentation](https://developer.hashicorp.com/terraform/language/resources/provisioners/syntax)