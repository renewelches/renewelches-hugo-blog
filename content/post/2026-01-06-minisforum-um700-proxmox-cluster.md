---
layout: post
title: "Expanding My Proxmox Cluster: Minisforum UM700 for $69"
subtitle: "Adding a refurbished mini PC to my homelab cluster"
date: 2026-01-06 14:00:00
author: "Rene Welches"
image: "/img/red-hook-crit.jpg"
publishDate: 2026-01-06 14:00:00
tags:
  - Proxmox
  - Homelab
  - Mini PC
  - Clustering
  - Hardware
categories: [homelab, Proxmox]
URL: "/2026/01/06/minisforum-um700-proxmox-cluster"
---

## The Deal of the Year

Last year I picked up a refurbished Minisforum UM700 mini PC Christmas deal, and the price was too good to pass up. Here's what I paid:

**Minisforum UM700 Refurbished**
- AMD Ryzen™ 7 3750H
- Barebone (no RAM/storage)
- Original price: $99.00
- Christmas discount: -$30.00
- **Total: $75.13 USD** inclduing taxes and free shipping

For ~ $75, I got a quad-core Ryzen 7 3750H (with SMT, so 8 threads) that's perfect for expanding my Proxmox homelab.

## Why Barebone Was Perfect

The UM700 came as a barebone system (no RAM or storage), which worked out perfectly because I had spare parts already:

- 16 GB DDR4 RAM from a previous upgrade and just bought another one used on ebay, which will hopefully come this week
- [Crucial P3 PCIe Gen3 NVMe 500GB SSD (Affiliate Link)](https://amzn.to/4jtkmPz) sitting unused in my parts drawer

Installing these took about 5 minutes—just pop off the top cover, slot in the RAM and NVMe drive, and reassemble. The UM700's tool-free design made this incredibly easy.

## Adding to My Proxmox Cluster

I already had a Proxmox node running on my [GMKtec NucBox M5 Ultra](https://www.gmktec.com/products/gmktec-nucbox-m5-ultra-amd-ryzen-7-7730u-mini-pc?srsltid=AfmBOoofnRD7PvufgSUlAt7I-XNUAx13LO7M5K4o86Fpw_c9ivj-AA6v) mini PC (`gmktec`). Adding the UM700 as a second node (`minis`) would give me:

- High availability for VMs
- Live migration capability
- Distributed resource management

Here's how I did it.

## Step 1: Enable Clustering on the Existing Node

Before you can join a new node to a cluster, you need to create the cluster on your existing node.

**Important:** You must enable clustering **before** attempting to join another node. Proxmox doesn't run in cluster mode by default.

SSH into your existing Proxmox node (`gmktec` in my case):

```bash
ssh root@gmktec
```

Create the cluster:

```bash
pvecm create homelab
```

Replace `homelab` with whatever name you want for your cluster.

Verify the cluster was created:

```bash
pvecm status
```

You should see output indicating:

- Cluster name: `homelab`
- Quorum information
- Node list showing your current node

## Step 2: Get the Cluster Join Information

**On the existing cluster node (`gmktec`)**, get the cluster join information via the web UI via `Datacenter -> Cluster -> Join Information`.

## Step 3: Join the New Node to the Cluster

SSH into your new Proxmox node (`minis`):

```bash
ssh root@minis
```

Join the cluster by pointing to the existing cluster node:

```bash
pvecm add gmktec
```

Replace gmktec with the hostname (or IP) of the existing cluster member.

You'll be prompted for:

- The root password of the existing node (`gmktec`)
- Confirmation to join the cluster

The command will:

1. Connect to the existing cluster node
2. Retrieve cluster configuration
3. Join this node to the cluster
4. Synchronize cluster database
5. Set up Corosync for cluster communication

This process takes about 30-60 seconds.

## Step 4: Verify the Cluster

Once the join completes, verify the cluster status from either node:

```bash
pvecm status
```

You should now see:

- **Name: homelab**
- Nodes: 2
- Quorum status showing "2 votes"
- and some more information

Check the cluster nodes list:

```bash
pvecm nodes
```

Output should show both nodes:

```bash
Membership information
----------------------
    Nodeid      Votes Name
         1          1 gmktec (local)
         2          1 minis
```

## What You Get with a Cluster

Now that both nodes are clustered, you can:

### 1. **Manage from One Interface**

Log into the Proxmox web UI on either node. You'll see both nodes in the datacenter view:

```
Datacenter (homelab)
  ├── gmktec
  └── minis
```

### 2. **Live Migration**

Migrate running VMs between nodes with zero downtime:

```bash
qm migrate <vmid> minis
```

### 3. **High Availability**

Configure VMs to automatically restart on another node if one fails:

- Go to **Datacenter → HA**
- Add VMs to the HA groups
- Set priorities and migration policies

### 4. **Shared Configuration**

- User accounts sync across nodes
- Network configuration is cluster-aware
- Storage can be shared (NFS, Ceph, etc.)

## Important Clustering Considerations

### Quorum and the Importance of Odd Numbers

With 2 nodes, you have a potential problem: if one node fails, you lose quorum (majority vote). The cluster will continue working but won't allow certain operations.

Solutions:

- **Add a third node** In the future I will add an old MINIX NEO J50C-4 Plus with 16GB DDR4/240GB as a 3rd node to fix the quorum problem and be able to setup a 3 node Kubernetes cluster.
- **Set expected votes** (for testing only): `pvecm expected 1`
- **Use a QDevice** for quorum arbitration (haven't tested this)
- Give one node 2 votes (haven't tested this)

### Network Configuration

Clustering requires:

- **Low latency network** between nodes (min. gigabit LAN)
- **Separate cluster network** (optional but recommended for production)

My setup uses a simple 5 port gigabit switch connecting both nodes directly to my router. Since I am running out of ports I just ordered a new [2.5GB Ethernet switch of Amazon (affiliate link)](https://amzn.to/4jpPsaK) with more ports.

### Storage Considerations

Each node currently has local storage. For live migration to work, you need shared storage:

- **NFS** (easy to set up)
- **Ceph** (distributed storage, requires 3+ nodes)
- **iSCSI** (centralized SAN)
- **GlusterFS** (distributed file system)

For now, I'm using local storage and doing offline migrations when needed.

## Performance of the UM700

The Ryzen 7 3750H is a surprisingly capable chip for homelab use:

- **4 cores / 8 threads** (great for multi-VM workloads)
- **Base clock: 2.3 GHz, Boost: 4.0 GHz**
- **35W TDP** (power-efficient, not as good as the Ryzen 7 7730U of the GMKtec with 15W TDP)
- **Vega 10 graphics** (hardware transcoding for Plex/Jellyfin)

## Command Reference

Here's a quick reference for Proxmox cluster commands:

```bash
# Create a cluster
pvecm create <cluster-name>

# Join a cluster (run on new node)
pvecm add <existing-node-ip-or-hostname>

# Check cluster status
pvecm status

# List cluster nodes
pvecm nodes

# Check expected votes (quorum)
pvecm expected

# Remove a node (run on the node to remove)
pvecm delnode <nodename>

# Migrate a VM (online migration)
qm migrate <vmid> <target-node>

# Migrate a VM (offline migration)
qm migrate <vmid> <target-node> --online 0
```

## Next Steps

Now that I have a 2-node cluster, I'm planning to:

- Add my MINIX NEO J50C-4 Plus, 16GB DDR4/240GB as a 3rd node and create a real Frankencluster 
- Set up NFS shared storage for VM templates and ISOs (maybe)
- Configure HA for critical VMs 
- Experiment with Ceph distributed storage (once I add a third node - maybe)

## Conclusion

For $75, I added a capable second node to my Proxmox cluster. The Minisforum UM700 is a fantastic deal for homelab enthusiasts, especially if you have spare RAM and storage lying around.

Proxmox's clustering is straightforward to set up—just remember to create the cluster on the first node before trying to join additional nodes. The command-line workflow is simple and well-documented.

If you're looking to expand your homelab without breaking the bank, refurbished mini PCs are the way to go. Keep an eye out for refurbrished deals on [Minisforum](https://refurbished.minisforum.com/).

Happy clustering!