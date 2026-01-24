---
layout: post
title: "Starting My Homelab Journey"
subtitle: "Building a Proxmox-based homelab with GMKtec NucBox M5 Ultra and setting up SSL certificates"
date: 2025-12-08 11:00:00
author: "Rene Welches"
description: "Starting a homelab with a $256 GMKtec NucBox M5 Ultra (Ryzen 7 7730U) and Proxmox VE. Includes setting up self-signed SSL certificates for secure HTTPS across all homelab services including Home Assistant."
image: "/img/red-hook-crit.jpg"
publishDate: 2025-12-08 11:00:00
tags:
  - Homelab
  - Proxmox
  - SSL
  - Mini PC
  - Home Assistant
categories: [Homelab, Proxmox]
URL: "/2025/12/08/homelab-beginning"
---

## Introduction

In the beginning there were two 16 GB DDR4 RAM sticks and a 1 TB M.2 SSD that were gathering dust. After researching mini PCs, I decided to purchase the [GMKtec NucBox M5 Ultra](https://www.gmktec.com/products/gmktec-nucbox-m5-ultra-amd-ryzen-7-7730u-mini-pc?srsltid=AfmBOoofnRD7PvufgSUlAt7I-XNUAx13LO7M5K4o86Fpw_c9ivj-AA6v) as a bare-bones system. The AMD Ryzen 7 7730U processor provides plenty of power for virtualization, and the compact form factor makes it perfect for a home lab setup. I got it on Black Friday sale for $255.89. Surprisingly, there were no import taxes added to the $255.89. The PC arrived from China after about two weeks.

**Specifications**

- **Processor**: AMD Ryzen 7 7730U
- **RAM**: 2x16 GB DDR4 (32 GB total - my existing hardware)
- **Storage**: 1TB M.2 SSD (also repurposed from my spare parts)
- **Form Factor**: Mini PC (perfect for a quiet homelab)

## Installing Proxmox VE

With the hardware assembled, the next step was to install [Proxmox Virtual Environment (VE)](https://www.proxmox.com/en/proxmox-virtual-environment/overview).

The installation was straightforward:

1. Downloaded the Proxmox VE ISO from the official website
2. Created a bootable USB drive
3. Booted the GMKtec NucBox from the USB drive
4. Followed the installation wizard

Within minutes, I had a fully functional hypervisor ready to host virtual machines and containers.
Update: After booting I recommend to run the [PVE Post Install](https://community-scripts.github.io/ProxmoxVE/scripts?id=post-pve-install) community script. It allows you to disbale the Enterprise repo enable the non-subscription repos when you are on non subscription version.

## Setting Up SSL Certificates

One of the first things I wanted to tackle was proper SSL/TLS certificate management. Running services over HTTPS is essential for security, even in a homelab environment. Additionally I will not have to set some Terrform `insecure` variable to accept self signed certificates. I documented the entire process in my GitHub repository: [homelab-self-signed-cert-setup](https://github.com/renewelches/homelab-self-signed-cert-setup).

The setup involved:

- Creating a Certificate Authority (CA) for my homelab
- Generating self-signed certificates for various services
- Configuring trust stores on client devices

## Securing Home Assistant

I also have a Raspberry Pi 4 running [Home Assistant](https://www.home-assistant.io/) for home automation. As part of my "security initiative", I configured TLS for the Home Assistant instance as well. This ensures that all communication with my smart home devices and automation platform is encrypted.

The process involved:

1. Generating SSL certificates for the Home Assistant domain
2. [Configuring Home Assistant to use HTTPS with self signed certicates](https://community.home-assistant.io/t/certificate-authority-and-self-signed-certificate-for-ssl-tls/196970)
3. Updating client applications to trust the new certificates e.g. an old Lenovo Tab 8 which is used as a home display

## What's Next?

Using Terraform to manage all my VMs on my new Proxmox server. I have another 16GB DDR4 RAM stick and another 500GB M2, maybe I'll add another serverd to the mix and setup a proxmox cluster...
