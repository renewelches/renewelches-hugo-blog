---
layout: post
title: "Homelab IaC: Terraform + Ansible for Proxmox LXC"
subtitle: "Because clicking through the Proxmox UI twice is already too many times"
date: 2026-04-20 10:00:00
author: "Rene Welches"
description: "How I went from Proxmox community scripts to a full Terraform + Ansible IaC setup for managing LXC containers, with MinIO as a self-hosted Terraform state backend."
image: "/img/proxmox-lxc-iac-banner.svg"
publishDate: 2026-04-20 10:00:00
tags:
  - Proxmox
  - Terraform
  - Ansible
  - Homelab
  - IaC
  - MinIO
  - LXC
categories: [Homelab]
URL: "/2026/04/20/proxmox-lxc-iac"
---

I have been tinkering with the best way to manage my homelab for a while now — specifically how to handle LXC containers on Proxmox in a way that doesn't feel like organized chaos. This post is about how I got from "just run the community script" to a proper Terraform + Ansible setup that I can actually maintain.

## The Community Scripts Phase

If you've spent any time in the Proxmox world, you've probably come across the [Proxmox community scripts](https://community-scripts.github.io/ProxmoxVE/). They are genuinely great for getting something running quickly or for kicking the tires on a new service. Spin up a container, run a script, done. Fast. Easy. Zero thinking required.

The problem is that zero thinking required also means zero repeatability, zero auditability, and zero IaC satisfaction. Coming from a background where infrastructure is code — not a series of bash scripts you ran once and hope you remember the options for — this was never going to stick for me long-term.

## The Pure Terraform Phase

So I went the other direction: pure Terraform with `remote-exec` provisioners. Provision the LXC, SSH in, install things. It worked fine for the initial setup. Terraform would create the container on Proxmox, run the provisioner, and you had a running service.

The cracks started showing the moment I wanted to *update* something. Terraform's `remote-exec` is designed for provisioning, not configuration management. Re-running it to change a config value or bump a container image isn't really what it's there for. I was essentially using a Swiss Army knife to do surgery.

## The Terraform + Ansible Setup

The current setup is a two-tier workflow that keeps concerns properly separated:

1. **Terraform** provisions the LXC containers on Proxmox and writes out an `inventory.ini` file.
2. **Ansible** picks up that inventory and deploys Docker containers into the LXC containers.

Keeping these two layers separate means I can re-provision infrastructure without touching service config, and re-deploy services without touching Proxmox. Each stack is fully independent with its own Terraform state, so I can deploy services à la carte — no need to apply the whole thing if I just want to spin up one new service.

The workflow is always the same:

```bash
# Step 1: Provision the LXC container
cd terraform/environments/prod/proxmox/<stack>
cp terraform.tfvars.example terraform.tfvars   # fill in IPs, credentials, etc.
terraform init
terraform apply

# Step 2: Deploy the service into the container
ansible-playbook -i ansible/inventory/prod/proxmox/<stack>/inventory.ini \
  ansible/deploy-<service>.yml
```

Terraform creates the container and writes the inventory. Ansible does the rest: installs Docker, pulls the images, configures TLS, starts the containers. Every deployed service runs behind TLS — I went all in on that decision early and it has saved a lot of headaches.

## The State Backend Problem

One thing that took some thought was where to store Terraform state. Working across two machines — laptop and desktop — makes local state files a real liability. I have personally felt the pain of a deleted or out-of-sync state file, and it's the kind of problem you only want to solve once.

The obvious answer is a remote state backend. The less obvious answer, when you're trying to keep things self-hosted, is MinIO.

### Why Not Just Use GCS or S3?

I could have pointed all my state at a GCP Storage bucket and called it a day. And in fact, I did exactly that for one state file — the MinIO state itself. More on that in a second. But for everything else, I wanted the state to live on my homelab, not in a cloud provider.

MinIO speaks the S3 API, which means Terraform's `s3` backend works with it out of the box with a few extra flags to skip AWS-specific validation:

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state"
    key    = "proxmox/<stack>/terraform.tfstate"
    region = "us-east-1"  # dummy value — MinIO ignores it

    endpoints = {
      s3 = "https://<your-minio-ip>:9000"
    }

    access_key = "<your-access-key>"
    secret_key = "<your-secret-key>"

    use_path_style              = true
    skip_credentials_validation = true
    skip_metadata_api_check     = true
    skip_region_validation      = true
    skip_requesting_account_id  = true

    use_lockfile = true
  }
}
```

State locking comes for free via MinIO's versioning. Create the bucket, enable versioning, and you've got a self-hosted backend that keeps multiple machines in sync and doesn't silently lose your state when you accidentally `rm -rf` the wrong folder.

### The MinIO Bootstrap Problem

There's one catch: you can't use MinIO as your Terraform backend before MinIO exists. So MinIO itself is deployed using a local Terraform state backend first. Once it's up, all other stacks use MinIO. The MinIO stack's own state lives in a GCP Storage bucket — a one-time exception I was willing to make to avoid the chicken-and-egg problem without overcomplicating things.

## Available Stacks

The repo currently supports the following services, each as an independent deployable stack:

| Stack | What it does |
|---|---|
| `minio` | S3-compatible object storage — deploy this first |
| `openwebui-searxng` | Open WebUI (LLM frontend) + SearXNG (private search) |
| `prometheus-grafana` | Prometheus metrics + Grafana dashboards |
| `forgejo` | Self-hosted Git service (HTTPS + SSH) |
| `n8n` | Workflow automation platform |
| `termix` | Web-based terminal manager |
| `claude-code` | Claude Code CLI environment |

More stacks will get added over time as I move more services under IaC management.

### A Note on Certificates

Every stack that exposes a web interface runs TLS. The Ansible playbooks expect the cert and key files to be present in `ansible/files/<stack>/` before you run them — they won't generate certs for you. If you want self-signed certificates for your homelab, I have a [separate repo for that](https://github.com/renewelches/homelab-self-signed-cert-setup).

One gotcha worth mentioning: if you're using self-signed certs, make sure you install the root CA on your host machines and configure Ansible to mount it into the LXC. The `openwebui-searxng` stack in particular needs this because Open WebUI talks to SearXNG over TLS — without the root CA trusted, web search in Open WebUI will silently fail.

## What's Next

The repo is at [github.com/renewelches/proxmox-lxc-iac](https://github.com/renewelches/proxmox-lxc-iac) if you want to take a look. I'll be adding more stacks as I continue migrating services, and I have a few ideas for making the TLS setup less manual. But that's a problem for future me.
