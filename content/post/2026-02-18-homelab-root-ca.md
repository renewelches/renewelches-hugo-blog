---
layout: post
title: "Running Your Own Root CA for the Homelab"
subtitle: "What started as a GitHub README turned into a proper blog post"
date: 2026-02-18 10:00:00
author: "Rene Welches"
description: "How to create a self-signed Root CA for your homelab, sign server certificates, and trust them on macOS and Linux — including the git gotcha that the macOS Keychain won't tell you about."
image: "/img/proxmox-homelab-banner.svg"
publishDate: 2026-02-18 10:00:00
tags:
  - Homelab
  - Security
  - TLS
  - macOS
  - Linux
categories: [homelab, security]
URL: "/2026/02/18/homelab-root-ca"
---

## It Started as a GitHub README

When I first set up a custom Root CA for my homelab I figured it didn't warrant a full blog post. I threw the steps into a GitHub repo — [homelab-self-signed-cert-setup](https://github.com/renewelches/homelab-self-signed-cert-setup) — and called it done.

Then the macOS Keychain bit me, and I spent longer debugging a `git push` failure than I care to admit. That's what turned this into a blog post.

## Why Bother With a Root CA?

Once your homelab grows beyond a couple of services, you start running things under internal domain names — Proxmox, Forgejo, Home Assistant, Grafana, and so on. Browsers and tools complain about self-signed certs. Instead of clicking through warnings forever, you can create your own Certificate Authority, trust it once on each machine, and sign as many internal certs as you want.

## Step 1: Install OpenSSL (macOS)

The version of OpenSSL bundled with macOS is actually LibreSSL, which works for most things but can behave differently. Install the real thing via Homebrew:

```bash
brew install openssl
```

## Step 2: Generate the CA Private Key

```bash
openssl genrsa -aes256 -out homelab-ca-private_key.pem 2048
```

This creates a 2048-bit RSA private key encrypted with AES-256. You'll be prompted for a passphrase — use a strong one and store it somewhere safe (a password manager). Every time you sign a new certificate you'll need it.

Flags:
- `-aes256` — encrypts the key with your passphrase
- `2048` — key size in bits (adequate for homelab use)

## Step 3: Create the Self-Signed Root CA Certificate

```bash
openssl req -x509 -new -nodes -key homelab-ca-private_key.pem \
  -sha256 -days 3650 -out homelab-root-CA.crt -subj "/CN=Home Lab CA"
```

This produces `homelab-root-CA.crt`, valid for 10 years. This is the file you'll distribute to every machine that needs to trust your internal certificates.

## Step 4: Sign a Server Certificate

Here's the workflow for signing a certificate for a specific service — I'll use Proxmox as the example.

### 4a. Generate a private key on the server

```bash
openssl genrsa -out proxmox.key 2048
```

### 4b. Create a config file with Subject Alternative Names

Modern TLS requires SANs — a bare CN is no longer sufficient. Create `proxmox.cnf`:

```ini
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = distinguished_name

[distinguished_name]
C = US
ST = New York
L = New York
O = home lab
OU = Proxmox
CN = proxmox.homelab.home, 192.168.1.10

[v3_req]
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = proxmox.homelab.home
IP.1 = 192.168.1.10
```

Adjust the DNS and IP entries to match your environment.

### 4c. Generate the Certificate Signing Request

```bash
openssl req -new -key proxmox.key -out proxmox.csr -config proxmox.cnf
```

### 4d. Copy the CSR to the machine holding your CA keys

```bash
scp proxmox.csr proxmox.cnf your-ca-machine:~/
```

Keep `proxmox.key` on the Proxmox server — it should never leave.

### 4e. Sign the CSR with your Root CA

```bash
openssl x509 -req -in proxmox.csr -CA homelab-root-CA.crt \
  -CAkey homelab-ca-private_key.pem \
  -CAcreateserial -out proxmox.crt -days 365 -sha256 \
  -extfile proxmox.cnf -extensions v3_req
```

### 4f. Verify the SANs are present

```bash
openssl x509 -in proxmox.crt -text -noout | grep "Subject Alternative Name" -A 1
```

Copy the resulting `proxmox.crt` back to the Proxmox node. You can upload it via the Proxmox web UI under **Node → Certificates → Upload Custom Certificate**.

## Step 5: Trust the Root CA

### Linux (Debian/Ubuntu)

```bash
sudo cp homelab-root-CA.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates --fresh
```

This is also needed if your Debian cloud-init VMs need to pull from internal services — I mentioned this in my [Proxmox cloud image template post](/2026/01/23/proxmox-debian-cloud-image/).

### macOS — and Where It Gets Tricky

```bash
sudo security add-trusted-cert -d -r trustRoot \
    -k /Library/Keychains/System.keychain \
    /Users/rene/Documents/Workspace/homelab-certificates/homelab-root-CA.crt
```

After running this, Keychain Access shows the certificate with **"Always Trust"** on every entry. Safari and Chrome pick it up correctly.

But `git push` to my internal Forgejo instance kept failing with a certificate verification error. The Keychain said it was trusted. Git disagreed.

## The macOS Keychain + Git Problem

It turns out Git on macOS does **not** use the system Keychain for TLS verification by default — it uses its own bundled CA bundle (from the curl or OpenSSL it was compiled against). The Keychain trust setting is irrelevant to it.

The fix is to tell Git explicitly where to find your root CA for requests to that specific host:

```bash
git config --global http.https://forgejo.grumples.home.sslCAInfo \
    /Users/rene/Documents/Workspace/homelab-certificates/homelab-root-CA.crt
```

This adds a scoped entry to your `~/.gitconfig` that applies only to that host:

```ini
[http "https://forgejo.grumples.home"]
    sslCAInfo = /Users/rene/Documents/Workspace/homelab-certificates/homelab-root-CA.crt
```

After that, `git push`, `git pull`, and `git clone` all work without any certificate errors.

If you have multiple internal services using the same CA, add a line for each hostname. The scoped format keeps things tidy and avoids globally disabling certificate verification (which you should never do).

## Recap

| Step | Command / Tool |
|------|----------------|
| Generate CA key | `openssl genrsa -aes256` |
| Create Root CA cert | `openssl req -x509` |
| Sign a server cert | `openssl x509 -req` |
| Trust on Linux | `update-ca-certificates` |
| Trust on macOS (system) | `security add-trusted-cert` |
| Trust in Git (macOS) | `git config --global http.<url>.sslCAInfo` |

The full setup scripts live in the [homelab-self-signed-cert-setup](https://github.com/renewelches/homelab-self-signed-cert-setup) repo if you want a quick reference without the commentary.