---
layout: post
title: "Real IPs for k3s: MetalLB, VLANs, and Why I Had to Buy a Switch"
subtitle: "A k3s LoadBalancer turned into a network rebuild — managed switch, VLANs, OPNsense, and MetalLB in L2 mode"
date: 2026-06-12 10:00:00
author: "Rene Welches"
description: "How I gave my Proxmox k3s cluster a real LoadBalancer with MetalLB — and why getting there meant buying a managed switch, segmenting the network into VLANs, and working around the limits of Google Nest WiFi."
image: "/img/k3s-metallb-banner.svg"
publishDate: 2026-06-12 10:00:00
tags:
  - Kubernetes
  - k3s
  - MetalLB
  - Homelab
  - Networking
  - VLAN
  - Proxmox
  - OPNsense
  - Twingate
  - Terraform
  - Ansible
categories: [Homelab]
URL: "/2026/06/12/k3s-metallb-vlan-homelab"
---

I wanted two things:

- a k3s cluster on my Proxmox mini-PCs with three nodes and
- a `Service type=LoadBalancer` which actually gets a real, reachable IP — the way it would in the cloud.

The former was just getting three cheap mini PCs for my Frankencluster ([Morefine M8](/2026/02/17/morefine-m8-proxmox-node/), [Minisforum UM700](/2026/01/06/minisforum-um700-proxmox-cluster/) and [GMKtec NucBox M5](/2025/12/08/homelab-beginning/)).

The latter took me buying a managed switch, a VLAN redesign, a firewall VM, and a zero-trust overlay to get there. This post is the story of how a "just install MetalLB" afternoon turned into rebuilding the network underneath it — and why it was actually necessary.

## What MetalLB Even Does

On a cloud provider, when you create a `Service type=LoadBalancer`, the provider notices and hands you an external IP. On bare metal there is no provider, so that service sits on `<pending>` forever. MetalLB is the missing piece: it watches for LoadBalancer services and assigns them IPs from a pool you define, then makes those IPs answer on your network.

### Why Not ServiceLB / klipper-lb?

Fair question — k3s actually ships with its own built-in LoadBalancer controller called [ServiceLB](https://github.com/k3s-io/klipper-lb) (formerly klipper-lb). It's enabled by default, and for a small cluster it is genuinely the path of least resistance.

The way it works is the catch: ServiceLB does not assign a dedicated external IP. It just binds the service port on the **host network of every k3s node** and uses the node IPs as the "external" IPs. That means:

- You can only have one LoadBalancer service per port across the entire cluster — two services both wanting `:80` will collide, because every node only has one `:80`.
- The "external IP" is whatever your nodes' IPs already are. You can't hand out stable, distinct IPs that survive a node moving or going down.
- Once you want a half-dozen services exposed on standard ports, you are immediately reaching for hacks.

MetalLB sidesteps all of that by handing each service its own IP from a pool, and announcing it on the network so the IP follows the service rather than the node. That is what I wanted, so ServiceLB had to come off — step zero of installing MetalLB on k3s is turning ServiceLB off, otherwise the two fight over the same LoadBalancer events. The exact flip lives in the install section below.

### What MetalLB Actually Needs

The catch is the phrase "a pool you define." MetalLB needs a block of IP addresses that:

- live in the **same subnet as the cluster nodes**,
- are **routable** to whoever needs to reach the services, and
- are **not handed out by any DHCP server**, or you get two machines fighting over the same address.

On my old flat network, I could not satisfy any of those cleanly. Which is where the trouble started.

## The Real Problem: Google Nest WiFi

My home network ran on Google Nest WiFi. As a consumer mesh it is great. As the foundation for a homelab it is a brick wall, and I hit every side of it:

- **No VLANs.** There is no concept of network segmentation. Everything — phones, the TV, my Proxmox cluster, random IoT gadgets — sits on one flat `192.168.1.0/24`. I could not carve out an isolated lab segment, and I could not reserve a clean IP range for MetalLB without it colliding with whatever DHCP felt like assigning.
- **No DHCP control.** You cannot set the DHCP scope, you cannot reserve a range, you can barely make static reservations. So "set aside `.200–.220` for MetalLB and tell DHCP to stay off it" — the one thing MetalLB asks for — was simply not an option.
- **No static routes.** This is the big one. Nest cannot be told "the `10.0.0.0/24` network lives behind that other router." It only knows its own flat subnet and the internet. The moment I put anything behind a second router, nothing in the house could route to it.
- **No bridge mode that survives mesh.** The usual escape hatch — put the consumer router in bridge mode and let a real router do the routing — disables the mesh. Not an option when the mesh is what keeps the household online.

To be fair, there is one workaround I think would have actually worked for the IP-pool problem: Nest does let you shrink the DHCP range. Start DHCP at `192.168.1.21` and you free up `.2–.20` as a static block MetalLB could own, on the same flat network, without buying anything. (I never tested this end-to-end, so take it with a grain of salt — but on paper it should hold.) The reason I did not go that route is that it only fixes the smallest piece of the problem. The cluster, the household, the TV, and every random IoT gadget would still share one broadcast domain, with no way to keep them from seeing each other. The IP carve-out was easy; the network shape was the actual issue.

So the flat network was not just _inconvenient_ for Kubernetes — it actively prevented the two things I actually wanted: an isolated lab subnet, and a clean separation between household traffic and whatever experimental thing I was about to break next. The network had to change first.

## Step One: A Managed Switch

The cheapest way out of "one flat dumb network" is a managed switch that understands **VLANs** (802.1Q). I picked up a [XikeStor 8-port 2.5G managed switch](https://www.amazon.com/dp/B0CLK5R1PC) — small, fanless, cheap, and 2.5 Gigabit on every port, which is a nice bonus for the cluster's storage replication.

With a managed switch I could split the single physical network into two logical ones on the same wires:

- **VLAN 1 — Home** (`192.168.1.0/24`): phones, laptops, TV, the Nest mesh. Untouched.
- **VLAN 10 — Lab** (`10.0.0.0/24`): the Proxmox guests, k3s, MetalLB, the self-hosted services. Isolated.

The switch tags traffic onto the right VLAN per port, and now I had a clean, empty `10.0.0.0/24` to design however I wanted — including a tidy MetalLB pool with nothing else fighting for it.

(One XikeStor gotcha that cost me a reboot's worth of confusion: you have to explicitly **"Save Config"** in the web UI after every change, or it all reverts when the switch power-cycles.)

## Step Two: A Real Router for the Lab — OPNsense

A VLAN gives you an isolated segment, but something still has to route it, hand out DHCP, and firewall it. Nest can't, so the lab got its own router: **OPNsense**, running as a VM on the one cluster node that has two physical NICs.

- **WAN** leg sits on the home VLAN (it gets internet the same way any other device does).
- **LAN** leg is the gateway for the lab, `10.0.0.1` — it does DHCP for `10.0.0.0/24`, NATs the lab out to the internet, and firewalls between lab and home.

DNS for the lab is **AdGuard Home** on a Raspberry Pi, serving a `homelab.home` domain with rewrites pointing service names at their lab IPs. Backups land on a **Proxmox Backup Server** that also lives in the lab. So "set up MetalLB" quietly became "stand up a router, a DNS server, and a backup target," because that is what an isolated network actually needs to be useful.

### The trunk trick: keep the cluster reachable

Here is a subtlety that bit me. Two of my three Proxmox nodes only have a **single** NIC. If I moved that NIC wholesale into the lab VLAN, the nodes themselves — Proxmox management, the cluster heartbeat (corosync) — would vanish from the home network and I would lock myself out.

The fix is to make those ports **trunks**: the node's management stays untagged on VLAN 1, while the VMs and containers on it ride VLAN 10 via a tagged sub-interface (`vmbr0.10`). One cable, two VLANs. Proxmox management and corosync never move — only the guests cross into the lab. The node with two NICs (GMKtec NucBox M5) runs OPNsense directly across both. The principle: segment the _workloads_, not the _hosts_.

```
                       internet
                           │
                 [ Google Nest WiFi ]   192.168.1.0/24  (VLAN 1, home)
                           │
        ┌──────────────────┴────────────────────┐
        │       XikeStor managed 2.5G switch    │
        │       carries VLAN 1 + VLAN 10        │
        └──┬───────┬──────────┬──────────┬──────┘
     trunk │  trunk│   VLAN 1 │  VLAN 10 │
   (1u+10t)│ (1u+10t) untagged│  untagged│
           │       │          │          │
       ┌───┴──┐ ┌──┴───┐    ┌─┴──────────┴──┐
       │ node2│ │ node3│    │     node1     │
       │ 1 NIC│ │ 1 NIC│    │     2 NICs    │
       │ mgmt │ │ mgmt │    │  WAN     LAN  │
       │  +   │ │  +   │    │     runs      │
       │ VMs  │ │ VMs  │    │   OPNsense    │
       │vmbr0 │ │vmbr0 │    │  LAN gateway  │
       │  .10 │ │  .10 │    │   10.0.0.1    │
       └───┬──┘ └───┬──┘    └───────┬───────┘
           │        │               │
           └────────┼───────────────┘
                    ▼
           Lab VLAN 10 — 10.0.0.0/24
           k3s VMs · MetalLB pool 10.0.0.200-220
```

## Step Three: Routing Home → Lab Without a Route — Twingate

Now I had a beautifully isolated lab… that nothing in the house could reach, because — remember — **Nest cannot do static routes**. From a laptop on the home VLAN, `10.0.0.x` was a black hole.

The clean answer would be a router at the home edge that knows the lab route. Nest isn't it, and I wasn't ready to rip out the mesh the whole household depends on. So instead of routing, I use **[Twingate](https://www.twingate.com/)** — a zero-trust overlay. A pair of lightweight connectors run as dual-homed containers (one foot in each VLAN), and my devices reach lab services through Twingate's tunnel. No exposed ports, no VPN server, and it solves the same problem on my phone and on the road, not just at home. I [wrote about the DNS side of that setup earlier](/2026/01/29/twingate-DNS-update/) — getting split-horizon names to resolve correctly through the tunnel is its own small adventure.

It is a workaround for a router limitation, and I am at peace with that. The day I replace the Nest with something that can actually route, Twingate becomes optional instead of load-bearing.

## Step Four: The k3s Cluster Itself

With the network sorted, the cluster is the easy part. Three VMs, provisioned with Terraform against the [bpg/proxmox provider](https://github.com/bpg/terraform-provider-proxmox), then k3s installed by Ansible:

- `k3s-server` → `10.0.0.10`
- `k3s-agent-1` → `10.0.0.11`
- `k3s-agent-2` → `10.0.0.12`

The one networking detail that matters for IaC: each VM has to land on the right bridge and VLAN. The node running OPNsense reaches the lab directly on its second NIC (untagged), while the single-NIC nodes need their VM tagged onto VLAN 10. I made that a per-node map in Terraform so the same module works whether you run VLANs or a flat network:

```hcl
# terraform.tfvars — per-node bridge + VLAN tag
k3s_bridges = {
  k3s_server = "vmbr1"   # node with 2 NICs: untagged lab bridge
  k3s_agent1 = "vmbr0"   # single-NIC node: trunk...
  k3s_agent2 = "vmbr0"
}
k3s_vlan_tags = {
  k3s_agent1 = 10        # ...tagged onto VLAN 10
  k3s_agent2 = 10
}
```

The defaults are a flat `vmbr0` with no tag, so anyone without VLANs can use the same code untouched — the segmentation is opt-in. The repo is on GitHub: [`proxmox-k3s-iac`](https://github.com/renewelches/proxmox-k3s-iac).

Crucially, the MetalLB pool — `10.0.0.200–220` — sits comfortably above the node IPs and outside the OPNsense DHCP scope. The thing I couldn't build on the Nest's flat network is now trivial on the lab VLAN.

## Step Five: Actually Installing MetalLB

Two components run in the `metallb-system` namespace: a **controller** (one pod, assigns IPs) and a **speaker** (a DaemonSet, one pod per node, makes the IPs answer on the wire). Installing it is one manifest. But on k3s there is a trap first.

### Disable the bundled ServiceLB

As covered earlier, k3s ships ServiceLB by default and it will fight MetalLB for every LoadBalancer service if both run side by side. Turn it off in the k3s config and restart:

```yaml
# /etc/rancher/k3s/config.yaml
disable:
  - servicelb
```

### Install, then configure the pool

With Klipper gone, apply the upstream manifest, wait for the controller's webhook to come up, then apply the pool. That ordering matters — the `IPAddressPool` is validated by a webhook that lives _inside_ the controller you just installed, so applying it too early fails. I wrapped the whole thing in an Ansible playbook with an explicit wait and a retry, but the essence is:

```yaml
# IPAddressPool + L2Advertisement
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.0.200-10.0.0.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default-pool-l2
  namespace: metallb-system
spec:
  ipAddressPools:
    - default-pool
```

That is it. Create a LoadBalancer service now and it comes up with a `10.0.0.2xx` address instead of `<pending>`. Even k3s's bundled Traefik ingress, which had been sitting pending, immediately grabbed a pool IP.

## Why L2 Mode (and Not BGP)

MetalLB has two modes, and the choice is really a question about your _router_, not your cluster.

- **BGP mode** is the "proper" datacenter approach: MetalLB peers with your router over BGP and advertises each service IP as a route, with real per-connection load balancing across nodes. It needs a router that speaks BGP.
- **L2 mode** needs nothing from the router at all. The speaker simply answers **ARP** for the pool IPs — it tells the local network segment "`10.0.0.205`? That's my MAC address," and traffic for that IP arrives at that node, where kube-proxy takes over.

For me the decision was made three routers ago. My home gateway is a Google Nest WiFi — it does not speak BGP, it does not speak static routes, it barely speaks DHCP reservations. BGP was never on the table. And honestly, even with OPNsense in the mix, L2 is the right call for a homelab this size:

- **It just works in a flat L2 segment.** Everything in the lab is one broadcast domain on VLAN 10. ARP is local. No peering, no route maps, no router config at all.
- **The trade-off is fine here.** L2 mode is failover, not true load balancing: each pool IP is "owned" by exactly one node at a time, and if that node dies another speaker takes over. The traffic for a given service funnels through one node. At three nodes and a handful of services, I will never notice. If I were running racks and needed every node pulling its weight, BGP would earn its complexity.

The one hard requirement L2 _does_ impose is the one I spent four sections earning: the pool has to be in the **same subnet as the nodes** (`10.0.0.x`) and **outside DHCP**. On the old flat Nest network I couldn't have guaranteed either. On the lab VLAN, both are free.

## The Gotchas, Collected

A few things that cost me time, in case they save you some:

- **"No cluster" was a lie my local state told me.** My Terraform state lives in a MinIO bucket, not on disk. I glanced at a stale local `terraform.tfstate` left over from before the backend migration, saw zero resources, and briefly convinced myself the cluster didn't exist. `terraform state list` (which reads the _real_ backend) set me straight. Trust the backend, not a leftover file — or better, clean up the stale files before they trick you.
- **The webhook race.** Applying the `IPAddressPool` immediately after the MetalLB manifest fails, because its validating webhook isn't up yet. Wait for the controller's rollout to finish first, then apply — with a retry for good measure.
- **Reaching the lab from my workstation.** The home VLAN has no route to `10.0.0.0/24` — Nest can't be told about it, so a direct SSH from my laptop to a lab node just times out. The fix lives in `ansible.cfg`: SSH `ProxyJump` through the one Proxmox node that has a leg in both VLANs, and every Ansible connection to `10.0.0.x` gets bounced through it transparently.

  ```ini
  [ssh_connection]
  ssh_args = -o ProxyJump=root@<dual-nic-node-home-ip>
  ```

  I worked this out after the first deploy hung on every task and I blamed the firewall — it was just the missing hop.

- **Save the switch config.** Worth repeating: the managed switch silently discards unsaved changes on reboot.

## Where This Lands

What started as "install MetalLB" turned into a genuine network: a managed switch splitting home from lab, OPNsense routing and firewalling the lab, AdGuard doing DNS, Twingate bridging the gap the Nest can't, a PBS server catching backups, and a k3s cluster sitting on top with real LoadBalancer IPs from a pool that nothing else can touch.

The irony is that MetalLB itself was the trivial part. The real work was building a network underneath it that could give it what it asked for — a clean subnet and a reserved range — neither of which a consumer mesh router will ever hand you. If you are staring at a `<pending>` LoadBalancer on a flat home network, the answer probably isn't more Kubernetes YAML. It's a managed switch.

Should you do the full rebuild, or try the "start DHCP at `192.168.1.21`" shortcut instead? It depends. If you want to learn about VLANs, the rebuild is the better path — you end up with a network you can grow into. If you just want a quick win and a working LoadBalancer, try the DHCP-range shortcut first; it's a fraction of the work for most of the result.

Next up: putting actual workloads on the cluster and wiring an ingress in front of them — now that there are IPs to point at.
