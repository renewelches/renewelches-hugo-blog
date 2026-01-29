---
layout: post
title: "Using Vagrant with QEMU on macOS - or not"
subtitle: "Lightweight VM provisioning on Apple Silicon with cloud-init support"
date: 2026-01-29 14:00:00
author: "Rene Welches"
description: "Learn how to set up Vagrant with QEMU on macOS using Homebrew. Explore two provisioning approaches: shell scripts and cloud-init, along with the advantages and limitations of this setup."
image: "/img/proxmox-homelab-banner.svg"
publishDate: 2026-01-29 14:00:00
tags:
  - Vagrant
  - QEMU
  - macOS
  - Cloud-Init
  - Virtualization
  - Apple Silicon
categories: [homelab, development]
URL: "/2026/01/29/vagrant-qemu-mac"
---

## Overview

Vagrant is a powerful tool for creating reproducible development environments. While VirtualBox has been the traditional provider on macOS, QEMU offers a compelling alternative, especially for Apple Silicon Macs where VirtualBox support is limited or non-existent.

This guide covers setting up Vagrant with QEMU on macOS using Homebrew, and demonstrates two approaches to VM provisioning: traditional shell scripts and cloud-init.

**Note:** While containerization tools like Docker Desktop or Colima are popular on macOS, they are not always a suitable alternative. When you need a full-blown VM—for testing kernel modules, running systemd-based services, simulating multi-node clusters, working with operating systems other than Linux, or locally testing configuration management tools like Ansible—containers simply won't cut it. QEMU with Vagrant fills this gap nicely on Apple Silicon (or maybe not always).

## Prerequisites

- macOS (Intel or Apple Silicon)
- Homebrew installed
- Basic familiarity with the command line

## Installation

### Step 1: Install QEMU

QEMU is an open-source machine emulator and virtualizer. Install it via Homebrew:

```bash
brew install qemu
```

### Step 2: Install Vagrant with the vagrant-qemu Plugin

Install Vagrant using Homebrew:

```bash
brew install --cask vagrant
vagrant plugin install vagrant-qemu
```

## Understanding the Vagrantfile Structure

A Vagrantfile defines your VM configuration. With the QEMU provider, you'll configure machine resources, SSH ports, and port forwarding differently than with VirtualBox.

### Key QEMU Provider Options

| Option                    | Description                  | Example                 |
| ------------------------- | ---------------------------- | ----------------------- |
| `qe.memory`               | RAM allocation in MB         | `1024`                  |
| `qe.cpus`                 | Number of CPU cores          | `2`                     |
| `qe.machine_virtual_size` | Disk size in GB              | `50`                    |
| `qe.ssh_port`             | Host SSH port for this VM    | `2223`                  |
| `qe.extra_netdev_args`    | Additional network arguments | `hostfwd=tcp::8081-:80` |

## Example 1: Shell Provisioning (SearXNG)

This example creates a VM for running SearXNG, a privacy-respecting metasearch engine, using traditional shell provisioning:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "searxng" do |searxng|
    searxng.vm.box = "cloud-image/debian-13"
    searxng.vm.synced_folder ".", "/vagrant", disabled: true
    searxng.ssh.insert_key = false
    searxng.vm.hostname = "searxng"

    searxng.vm.provider "qemu" do |qe|
      qe.memory = 1024
      qe.cpus = 1
      qe.machine_virtual_size = 50

      # SSH port (unique per VM to allow parallel running)
      qe.ssh_port = 2223

      # Additional port forwarding for SearXNG (host:8081 -> guest:80)
      qe.extra_netdev_args = "hostfwd=tcp::8081-:80"
    end

    # Provision Docker via shell
    searxng.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y docker.io docker-compose
      systemctl enable docker
      systemctl start docker
      usermod -aG docker vagrant
    SHELL
  end
end
```

### Key Configuration Points

- **Box**: Uses `cloud-image/debian-13`, a cloud-optimized Debian image with QEMU support
- **Synced Folder**: Disabled to avoid QEMU compatibility issues
- **SSH Key**: `insert_key = false` keeps the insecure Vagrant key for simplicity
- **Port Forwarding**: Maps host port 8081 to guest port 80 via `extra_netdev_args`
- **Shell Provisioning**: Installs Docker after the VM boots via shell

## Example 2: Cloud-Init Provisioning (n8n)

This example creates a VM for n8n, a workflow automation tool, using cloud-init for provisioning:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "n8n" do |n8n|
    n8n.vm.box = "cloud-image/debian-13"
    n8n.vm.synced_folder ".", "/vagrant", disabled: true
    n8n.ssh.insert_key = false
    n8n.vm.hostname = "n8n"

    n8n.vm.cloud_init do |cloud_init|
      cloud_init.content_type = "text/cloud-config"
      cloud_init.inline = <<-EOF
        package_update: true
        package_upgrade: true
        packages:
          - docker.io
          - docker-compose
        runcmd:
          - systemctl enable docker
          - [ systemctl, start, --no-block, docker.service ]
          - usermod -aG docker vagrant

      EOF
    end

    n8n.vm.provider "qemu" do |qe|
      qe.memory = 6144
      qe.cpus = 2
      qe.machine_virtual_size = 50

      # SSH port (unique per VM to allow parallel running)
      qe.ssh_port = 2224

      # Additional port forwarding for n8n (host:5678 -> guest:5678)
      qe.extra_netdev_args = "hostfwd=tcp::5678-:5678"
    end
  end
end
```

### Cloud-Init vs Shell Provisioning

| Aspect           | Shell Provisioning                  | Cloud-Init                    |
| ---------------- | ----------------------------------- | ----------------------------- |
| Execution timing | After VM boots and SSH is available | During initial boot process   |
| Declarative      | No (imperative scripts)             | Yes (YAML configuration)      |
| Package handling | Manual `apt-get` commands           | Built-in `packages` directive |
| Idempotency      | Must be manually ensured            | Handled automatically         |
| Familiarity      | Standard shell scripting            | Requires cloud-init knowledge |

Cloud-init is particularly useful when:

- You want the same provisioning approach as your cloud or Proxmox deployments
- You need the VM ready immediately after first boot
- You prefer declarative configuration over scripts

## Running the VMs

### Basic Commands

```bash
# Start the VM
vagrant up --provider=qemu

# SSH into the VM
vagrant ssh

# Check VM status
vagrant status

# Stop the VM
vagrant halt

# Destroy the VM
vagrant destroy
```

### Running Multiple VMs

When running multiple QEMU VMs, ensure each has a unique SSH port:

```ruby
# VM 1
qe.ssh_port = 2223

# VM 2
qe.ssh_port = 2224
```

## Advantages of Using QEMU on macOS

### 1. Apple Silicon Support

QEMU runs natively on Apple Silicon Macs, providing near-native performance through hardware virtualization (Hypervisor.framework). VirtualBox still lacks full Apple Silicon support.

### 2. Lightweight

QEMU has a smaller footprint compared to VirtualBox or VMware Fusion.

### 3. Cloud Image Compatibility

QEMU works seamlessly with cloud images (qcow2 format), making it easy to use the same images you'd deploy to Proxmox or cloud providers.

### 4. Cloud-Init Integration

Native support for cloud-init allows you to develop and test your cloud-init configurations locally before deploying to production.

### 5. Cross-Architecture Emulation

While slower, QEMU can emulate different CPU architectures (x86 on ARM and vice versa) when needed for testing.

## Limitations and Disadvantages

### 1. No Virtual Network Support

This is the most significant limitation. When starting a VM, you'll see:

```
==> searxng: Warning! The QEMU provider doesn't support any of the Vagrant
==> searxng: high-level network configurations (`config.vm.network`). They
==> searxng: will be silently ignored.
```

This means:

- **No private networks**: You cannot create isolated networks between VMs
- **No multi-VM communication**: VMs cannot directly communicate with each other using private IPs
- **Port forwarding only**: The only networking option is forwarding ports from host to guest

If you need to test multi-tier applications with separate VMs communicating over a private network, you'll need to use a different provider or workaround (like having services communicate through the host machine's forwarded ports - which I haven't tested).

### 2. Limited Vagrant Feature Support

Some Vagrant features that work with VirtualBox may not work with QEMU:

- Synced folders require extra configuration or may not work at all
- Network configurations are limited to port forwarding
- Some box images may not be QEMU-compatible

### 3. Fewer Available Boxes

While the `cloud-image/*` boxes work well, the selection of QEMU-compatible Vagrant boxes is smaller than VirtualBox.

### 4. Manual Port Management

You must manually ensure SSH ports and forwarded ports don't conflict between VMs.

## Workarounds for Network Limitations

If you need multi-VM communication, consider these alternatives:

### 1. Use Host as Router

Forward ports for each service and have VMs communicate through the host's IP address.

### 2. Container Networking

Run containers inside a single VM and use Docker's networking capabilities for inter-service communication.

### 3. Use UTM

For GUI-based VM management with better networking on Apple Silicon, consider [UTM](https://mac.getutm.app/), which provides a more feature-rich QEMU frontend.

## Conclusion

Vagrant with QEMU is an excellent choice for macOS users, particularly those on Apple Silicon who need lightweight, reproducible development environments. The combination of cloud-init support and cloud image compatibility makes it ideal for developing configurations that mirror production deployments on Proxmox or cloud providers.

However, the lack of virtual networking support is a notable limitation. For simple single-VM development environments or testing cloud-init configurations, QEMU works wonderfully. For complex multi-VM setups requiring inter-VM communication, you may need to explore alternative solutions or providers. I'll have to give UTM or VirtualBox with Vagrant a try and see how that works.
