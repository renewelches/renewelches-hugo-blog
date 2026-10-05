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

> **Updated October 5, 2026:** The original Root CA from this post was missing the `keyUsage` extension, which Python 3.13+ rejects. Steps 3 and 4b are corrected, and there is a new section at the end on [what changed and how to fix an existing CA](#update-2026-10), including a note on Apple's certificate validity limits.

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

First create a small config file, `homelab-ca.cnf`, that spells out the CA extensions explicitly:

```ini
[req]
prompt = no
default_md = sha256
distinguished_name = distinguished_name
x509_extensions = v3_ca

[distinguished_name]
CN = Home Lab CA

[v3_ca]
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
```

Then create the certificate:

```bash
openssl req -x509 -new -key homelab-ca-private_key.pem \
  -sha256 -days 3650 -out homelab-root-CA.crt -config homelab-ca.cnf
```

This produces `homelab-root-CA.crt`, valid for 10 years. This is the file you'll distribute to every machine that needs to trust your internal certificates.

Check that the extensions made it in:

```bash
openssl x509 -in homelab-root-CA.crt -noout -text | grep -A1 "X509v3"
```

You want to see `Basic Constraints: critical` with `CA:TRUE` and `Key Usage: critical` with `Certificate Sign, CRL Sign`.

*Update (October 2026): the original version of this step was a one-liner with `-subj "/CN=Home Lab CA"` and no config file. That relies on whatever defaults your OpenSSL ships with, and those defaults do not include `keyUsage`. See the [update at the end](#update-2026-10) for why that matters.*

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
basicConstraints = critical, CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid
subjectAltName = @alt_names

[alt_names]
DNS.1 = proxmox.homelab.home
IP.1 = 192.168.1.10
```

Adjust the DNS and IP entries to match your environment.

*Update (October 2026): `basicConstraints`, `subjectKeyIdentifier` and `authorityKeyIdentifier` were added. Without the Authority Key Identifier, Python 3.13+ rejects the server certificate — details in the [update at the end](#update-2026-10).*

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

## Update, October 2026: Python 3.13 and Apple's Validity Limits {#update-2026-10}

Two things came up after I published this post. One was a real bug in my instructions, the other was me half-remembering an Apple rule.

### Python 3.13 rejects the original Root CA

Browsers, curl and git were all happy with the CA from the original Step 3. Python was not. Anything using `requests`, `httpx` or plain `urllib` against an internal service died with:

```text
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: CA cert does not include key usage extension
```

The reason: since Python 3.13, `ssl.create_default_context()` turns on `VERIFY_X509_STRICT` by default (see the [3.13 release notes](https://docs.python.org/3/whatsnew/3.13.html#ssl)). Strict mode enforces RFC 5280 properly, and RFC 5280 says a CA certificate must carry a `keyUsage` extension with `keyCertSign`. My one-liner never set it. OpenSSL's default config only adds `basicConstraints` and the key identifiers for you, so the CA came out without any key usage at all. Python 3.12 and older, curl and the browsers don't check this, which is why it went unnoticed for a while.

The fix is the explicit `[v3_ca]` section now shown in Step 3:

```ini
basicConstraints = critical, CA:TRUE
keyUsage = critical, keyCertSign, cRLSign
subjectKeyIdentifier = hash
```

There was a second, sneakier failure hiding behind the first one. Once the CA was fixed, a server certificate signed with a newer OpenSSL still failed in Python:

```text
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: Missing Authority Key Identifier
```

Strict mode also wants an Authority Key Identifier on the server certificate. Whether `openssl x509 -req` adds one on its own depends on the OpenSSL version: on my Mac, OpenSSL 3.0 added it and Homebrew's OpenSSL 4.0 did not. Same commands, different certificates. That is why Step 4b now sets `subjectKeyIdentifier` and `authorityKeyIdentifier` explicitly instead of trusting the defaults.

By the way, check which `openssl` you are actually running with `which -a openssl`. I had three on my Mac (Anaconda, Homebrew and the system LibreSSL) and the first one in the `PATH` was not the one I thought it was.

### Fixing an existing CA without starting over

You don't need a new CA key. Re-issue the Root CA certificate with the **same private key** and the new config:

```bash
openssl req -x509 -new -key homelab-ca-private_key.pem \
  -sha256 -days 3650 -out homelab-root-CA.crt -config homelab-ca.cnf
```

Because the key and the subject name stay the same, server certificates you already signed still chain to the re-issued CA. What you do have to redo:

- **Re-distribute and re-trust the new `homelab-root-CA.crt`** on every machine (Step 5), and remove the old one from the macOS Keychain so you don't end up with two "Home Lab CA" entries. It is a different certificate, even though the key is the same.
- **Re-sign server certificates that lack an Authority Key Identifier**, using the updated config from Step 4b. Check with `openssl x509 -in proxmox.crt -noout -text | grep "X509v3"`.

To test the whole chain the way Python 3.13 does:

```bash
openssl verify -x509_strict -CAfile homelab-root-CA.crt proxmox.crt
```

You can also switch strict mode off in Python with `ctx.verify_flags &= ~ssl.VERIFY_X509_STRICT`, but that only helps for code you control, and fixing the certificate is the better option.

### Apple's validity limits

I remembered that Safari wants certificates to be valid for less than a year or so. The actual rules are a bit different, and the good news is that the commands in this post were already fine:

- **398 days** is the limit Apple introduced for TLS server certificates issued on or after September 1, 2020, but only for certificates issued from the Root CAs that ship preinstalled with Apple's operating systems. Apple states explicitly that it does [not affect user-added or administrator-added Root CAs](https://support.apple.com/en-us/102028). So this one does not apply to a homelab CA. (For public CAs the limit has since dropped further, to 200 days as of March 2026.)
- **825 days** is the one that matters here. Since iOS 13 and macOS 10.15, TLS server certificates issued after July 1, 2019 must have a [validity period of 825 days or fewer](https://support.apple.com/en-us/103769). Apple's page lists no exemption for private CAs.

The same Apple page also requires the `serverAuth` Extended Key Usage, the DNS name in the Subject Alternative Name, RSA keys of at least 2048 bits and a SHA-2 signature. Step 4 covers all of those.

These limits are about the **server certificate**, not the Root CA. So 10 years (`-days 3650`) for the Root CA is fine, and `-days 365` for the server certificates in Step 4e is comfortably below 825. If you are tempted to sign a server certificate for 10 years so you never have to renew it: Safari will refuse it.

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