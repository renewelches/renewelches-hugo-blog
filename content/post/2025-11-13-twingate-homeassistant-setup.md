---
layout: post
title: "Secure Home Network Access with Twingate"
subtitle: "Alternative setup to Pi-Hole and Pi-VPN with Twingate, Home Assistant and AdGuard Home"
date: 2025-11-13 11:00:00
author: "Rene Welches"
image: "/img/red-hook-crit.jpg"
publishDate: 2025-11-13 11:00:00
tags:
  - Homelab
  - Security
  - Twingate
  - Home Assistant
  - DNS
  - VPN
categories: [Homelab]
URL: "/2025/11/13/twingate-homeassistant-setup"
---

## Introduction

Today, I had a chance to try out my new Twingate setup, and I noticed that my DNS wasn't working. In this guide, I'll show you how to set up Twingate to access your home network resources, including how to configure DNS using AdGuard Home on Home Assistant.

In the past, I used Pi-Hole (for ad blocking/DNS) and PiVPN (with WireGuard protocol), a Google domain, and [Inadyn](https://troglobit.com/projects/inadyn/) (DDNS client) for VPN. Everything was working great until Google decided to sell their DNS business to Squarespace (not a great move, Google). To make matters worse, Squarespace doesn't support DDNS. I was already considering switching my DNS to something like Cloudflare, but then I saw [NetworkChuck](https://www.youtube.com/watch?v=IYmXPF3XUwo) promoting Twingate and decided to give it a try. And I must say, I love it.

Twingate provides a modern zero-trust network access solution that's perfect for securely accessing your home network resources from anywhere. Unlike traditional VPNs, Twingate offers device-level authentication, split-tunneling by default, and doesn't require opening ports on your router or managing DDNS.

## Prerequisites

- A running Home Assistant instance on your local network
- [AdGuard Home](https://github.com/hassio-addons/addon-adguard-home)
- A Twingate account (free tier is sufficient)
- The Twingate Connector for Home Assistant (there are other options available as well)

## Part 1: Setting up AdGuard Home DNS

### Why Local DNS Matters

When accessing a resource like Home Assistant on your home network, you want to use a domain like `*.homelab.internal`. I ran into a snag with the recommended domain [home.arpa](https://www.rfc-editor.org/rfc/rfc8375.html) for home networks, and it looks like I'm not alone in this, [as mentioned on Reddit](https://www.reddit.com/r/MacOS/comments/1bu62do/homearpa_is_anyone_else_successfully_using_this/) and on [Medium](https://medium.com/@life-is-short-so-enjoy-it/homelab-adding-local-dns-entry-into-adguard-home-arpa-and-pushing-to-clients-from-udm-se-8493253830e5). Therefore, I opted for a different domain, such as `homelab.internal`, which has been functioning seamlessly across all devices.

### Setup AdGuard Home as DNS

1. **Access your Home Assistant instance** at Home Assistant's default location http://homeassistant.local:8123.
2. **Install the AdGuard Home Plugin** from the Home Assistant Add-on Store. Enable the `watchdog` option (recommended) and optionally `show in sidebar` (very convenient). Start the plugin.
3. **Set Up Upstream DNS Servers**

   - Navigate to **AdGuard Home Plugin** and then in the top navigation to **Settings** → **DNS Settings**.
   - In the `Upstream DNS servers` form, add your upstream DNS provider. For example:

   ```
     https://dns10.quad9.net/dns-query
     9.9.9.9

   ```

   - Scroll down and press `Apply`.

4. **Look up your Home Assistant’s IP address**
5. **Set up your Router's DNS Server**
   - Access your router's settings.
   - Find the DHCP or LAN settings.
   - Enter the IP address of your Home Assistant instance into the DNS server field.
6. **Add DNS Rewrite Rule**
   - Navigate to **AdGuard Home Plugin** and then in the top navigation to
     **Filters** → **Custom filtering rules**.
   - In the `Custom filtering rules` form, add your hosts like you would in your `/etc/hosts` file. See example below.
     ![Setup DNS rules](/img/homeassistant/AdGuard-DNS.png)
   - Save the rule by pressing `Apply`.
7. **Verify DNS Resolution**
   ```bash
   # From a Mac/Linux device on your local network
   nslookup homeassistant.homelab.local
   # Should return your Home Assistant IP
   ```

## Part 2: Install Twingate Connector

The Twingate Connector is a lightweight service that runs on your network and provides secure access to your resources. Since you already have Home Assistant running, the easiest option is to install it on the Home Assistant instance from the Add-on Store by following the [Twingate documentation](https://www.twingate.com/docs/home-assistant-getting-started).

## Part 3: Configure Twingate Resources

### Add Home Assistant as a Resource

1. **Navigate to Resources**

   - In Twingate Admin Console, go to **Network** → **Resources**
   - Click **Add Resource**

2. **Configure Resource Details**
   - **Name**: e.g., `Home Assistant`
   - **Address**: Use the same as previously used in the AdGuard configuration, e.g., `homeassistant.homelab.internal` or `192.168.86.11`, and use `Alias` to add `homeassistant.homelab.internal` as the DNS name.
   - **Optionally**, you can also configure the port for your Home Assistant (or other service) if needed.
3. If you set up a custom policy, you can assign it to the resource.

![Twingate Resource Creation](/img/homeassistant/Twingate-Resource.png)

## Part 4: Client Setup

### Install Twingate Client

1. **Download the client** for your platform:

   - Windows/Mac/Linux: [twingate.com/download](https://www.twingate.com/download)
   - iOS/Android: Search "Twingate" in app stores

2. **Authenticate**

   - Enter your network name (e.g., `homelab.twingate.com`)
   - Complete SSO or email authentication
   - Accept any required policies

3. **Verify DNS Setup**

   ```bash
   # From a Mac/Linux device on your local network
   nslookup homeassistant.homelab.local
   # Should return your Home Assistant IP
   ```
