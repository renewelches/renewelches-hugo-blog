---
layout: post
title: "Upgrading My "Frankenstein-cluster": MOREFINE M8 Replaces the MINIX"
subtitle: "A newer, faster node for $120 net after selling the old one"
date: 2026-02-17 12:00:00
author: "Rene Welches"
description: "I picked up a MOREFINE M8 mini PC (Intel N150, 16GB DDR4, 1TB NVMe) for $219.99 and sold my old MINIX NEO J50C-4 Plus on eBay for $100, netting a solid upgrade for just $120."
image: "/img/proxmox-homelab-banner.svg"
publishDate: 2026-02-17 12:00:00
tags:
  - Proxmox
  - Homelab
  - Mini PC
  - Hardware
categories: [homelab, Proxmox]
URL: "/2026/02/17/morefine-m8-proxmox-node"
---

## A New Addition to the Frankencluster

Back in January I [wrote about adding the Minisforum UM700 as a second Proxmox node](/2026/01/06/minisforum-um700-proxmox-cluster/) and mentioned the MINIX NEO J50C-4 Plus as a potential third node. That plan changed — instead of pressing the aging MINIX into service, I decided to sell it and put the money toward something newer and more capable.

## The Numbers

**Out:**

- MINIX NEO J50C-4 Plus, 16GB DDR4 / 240GB storage
- Sold on eBay for **$100**

**In:**

- MOREFINE M8 Mini PC — Intel N150, 16GB DDR4, 1TB NVMe SSD
- Purchased on Amazon for **$219.99**
- **Net cost: ~$120**

For $120 out of pocket I went from a Pentium Silver J5005 (Gemini Lake, 2018) with 240GB of storage to a brand-new Intel N150 with a full terabyte of NVMe. That felt like a good trade.

## Why the MOREFINE M8?

The Intel N150 is part of Intel's 13th gen "Twin Lake" series — a modern, energy-efficient chip designed exactly for always-on appliances and mini PCs. Key specs:

- **CPU:** Intel N150, 4 cores / 4 threads, up to 3.6 GHz boost
- **TDP:** 6W — runs cool and quiet without a fan upgrade
- **RAM:** 16GB DDR4 (soldered)
- **Storage:** 1TB NVMe SSD
- **Display:** 4K dual display output
- **Networking:** RJ45 Ethernet + Wi-Fi
- **USB:** USB 3.2 ports

Compared to the MINIX's Pentium Silver J5005, the N150 is a newer architecture with better single-threaded performance and significantly lower idle power draw — both important for a node that runs 24/7.

The 1TB NVMe is also a meaningful upgrade over the 240GB that was in the MINIX, giving me much more room for VM templates and local storage.

## Adding It to the Proxmox Cluster

Adding the M8 as a third node was straightforward using the same process I documented in the [UM700 cluster post](/2026/01/06/minisforum-um700-proxmox-cluster/):

1. Install Proxmox VE on the M8
2. Join the existing `homelab` cluster via `Datacenter -> Cluster -> Join Information`
3. Verify with `pvecm status`

With three nodes the cluster now has a proper quorum majority, which means it can handle a single node failure without the entire cluster going read-only. This was the main motivation for adding a third node — two nodes is a quorum tightrope.

## Was It Worth It?

Absolutely. The MINIX had served its purpose but wasn't doing much sitting on a shelf. Selling it for $100 and topping up $120 more got me:

- A modern, low-power CPU
- 4× the storage
- A real three-node cluster with proper quorum

The homelab Frankencluster now has three nodes: the **GMKtec NucBox M5 Ultra** (Ryzen 7 7730U), the **Minisforum UM700** (Ryzen 7 3750H), and the new **MOREFINE M8** (Intel N150). A nice mix of x86 hardware all running Proxmox VE.
