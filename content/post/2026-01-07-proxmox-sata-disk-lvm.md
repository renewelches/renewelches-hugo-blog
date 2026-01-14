---
layout: post
title: "Adding a Disk to Proxmox node: LVM Integration Guide"
subtitle: "Extend your Proxmox storage with LVM thin provisioning"
date: 2026-01-08 11:00:00
author: "Rene Welches"
description: "Complete guide to adding a disk to a Proxmox node with existing storage. Learn how to integrate the disk into the LVM 'pve' volume group and extend the thin pool for transparent storage expansion."
image: "/img/red-hook-crit.jpg"
publishDate: 2026-01-08 18:00:00
tags:
  - Proxmox
  - LVM
  - Storage
  - Tutorial
  - Linux
categories: [homelab, Proxmox]
URL: "/2026/01/08/proxmox-add-disk-lvm"
---

## Overview

This guide walks through adding a new disk to an existing Proxmox installation with NVMe storage, integrating it into the LVM `pve` volume group, and making it available to the thin pool used by Proxmox. In my case I added a 2nd SATA disk to the existing NVME disk on my minisforum um700 node. I had another [Crucial BX500 480GB (Affiliate Link)](https://amzn.to/49qZvb9) in my hardware box. With this addition my minisforum um700 storage is almost on par with the GMKtec NucBox M5 Ultra storage.

## Prerequisites

- Existing Proxmox installation with LVM `pve` volume group
- New SATA or other disk physically installed in your system
- Root access via SSH or console
- Basic understanding of LVM concepts (optional but helpful)

## What We're Doing

1. Prepare the new disk
2. Create an LVM physical volume from the disk
3. Add it to the existing `pve` volume group
4. Extend the thin pool to use the new space

## Step-by-Step Guide

### Step 1: Verify Your Current Setup

First, check your existing LVM configuration:

```bash
sudo pvs            # shows all physical volumes
sudo vgdisplay pve  # displays the virtual group 'pve' information
sudo lvs pve        # shows logical volumes information of the 'pve' group
```

Check Proxmox's storage configuration:

```bash
cat /etc/pve/storage.cfg
```

Look for an entry like:
```
lvmthin: local-lvm
    thinpool data
    vgname pve
    content rootdir,images
```

This confirms you're using LVM thin provisioning with a thin pool named `data`.

### Step 2: Prepare the New Disk

Identify your new disk (be careful here—use `lsblk` to verify):

```bash
lsblk
```

Look for your new disk. Let's assume it's `/dev/sda`. If your disk has an existing partition table or filesystems, clean it completely:

```bash
sudo wipefs -a /dev/sda
```

This erases all partition tables, filesystem signatures, and LVM metadata.

**Note:** Only run this if the disk is truly new and has no data you need.

### Step 3: Create an LVM Physical Volume

Convert the disk to an LVM physical volume:

```bash
sudo pvcreate /dev/sda
```

Verify it was created:

```bash
sudo pvs
```

You should see `/dev/sda` listed as a new physical volume.

### Step 4: Add to the pve Volume Group

Extend the existing `pve` volume group to include the new physical volume:

```bash
sudo vgextend pve /dev/sda
```

Verify the volume group grew:

```bash
sudo vgdisplay pve
```

Note the increased `VG Size` and `Alloc PE / Size` values.

### Step 5: Extend the Thin Pool

The thin pool `data` is what Proxmox actually uses. Extend it to allocate the new disk space:

```bash
sudo lvextend -l +100%FREE /dev/pve/data
```

This expands the thin pool to use all newly available space from `/dev/sda`.

Verify the thin pool expanded:

```bash
sudo lvs pve/data
```

The size should have increased significantly.

### Step 6: Verify in Proxmox

In the Proxmox web UI:
- Navigate to **Datacenter → Storage → local-lvm**
- You should see the increased capacity available

## How It Works

Once you complete these steps, Proxmox and LVM handle the complexity behind the scenes:

- **Distribution:** LVM automatically distributes data across `/dev/nvme0n1p3` and `/dev/sda` as needed
- **Transparent:** Proxmox sees one unified pool (`local-lvm`) regardless of underlying physical volumes
- **Flexible:** Add more disks in the future using the same process

## Alternative: Using Partitions Instead of Whole Disk

If you prefer to partition the disk first (for organizational purposes), you can:

```bash
sudo gdisk /dev/sda
# Create partitions as needed (e.g., /dev/sda1, /dev/sda2)
# Write and exit (w → y)

sudo pvcreate /dev/sda1 /dev/sda2
sudo vgextend pve /dev/sda1 /dev/sda2
sudo lvextend -l +100%FREE /dev/pve/data
```

However, for simplicity, using the entire disk as a single physical volume is recommended.

## Troubleshooting

### "Cannot use /dev/sda: device is partitioned"

The disk still has partition table metadata. Clean it:

```bash
sudo wipefs -a /dev/sda
```

Then retry `pvcreate`.

### LVM refuses to remove disk from volume group

If the disk is actively in use by the thin pool and you need to remove it:

```bash
sudo lvchange -an /dev/pve/data    # Deactivate
sudo vgreduce pve /dev/sda --force # Force remove
sudo lvchange -ay /dev/pve/data    # Reactivate
```

However, this is rarely needed. Usually, just physically moving a disk keeps it recognized by LVM.

## Key Takeaway

Adding storage to Proxmox with LVM is straightforward: create a physical volume, add it to the volume group, and extend the thin pool. LVM handles the complexity of distributing data across multiple disks transparently.
