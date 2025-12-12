---
layout: post
title: "Securing Proxmox API Tokens with Apple Keychain for Terraform"
subtitle: "Store and retrieve Proxmox credentials securely using macOS Keychain instead of plain text files"
date: 2025-12-09 14:00:00
author: "Rene Welches"
image: "/img/red-hook-crit.jpg"
publishDate: 2025-12-09 14:00:00
tags:
  - Homelab
  - Proxmox
  - Terraform
  - Security
  - macOS
categories: [Homelab]
URL: "/2025/12/09/proxmox-terraform-keychain"
---

## Introduction

When working with Terraform to manage Proxmox infrastructure, you need to authenticate. Proxmox offers three options: API Token, Auth Ticket, and Username/Password.

In my setup, I use API Tokens for authentication with Proxmox. The approach described here should also work with username/password combinations.

Storing these credentials in plain text files or environment variables poses security risks, especially if you accidentally commit them to version control. macOS provides a secure solution: the **Keychain Access**, which encrypts and manages passwords system-wide.

## Creating Proxmox Users and API Tokens

Before you can use API tokens with Terraform, you need to create a dedicated user and generate an API token in Proxmox. For detailed instructions, see the [official Proxmox documentation on User Management and API Tokens](https://pve.proxmox.com/pve-docs/chapter-pveum.html#pveum_tokens).

**Quick steps:**

1. Create a new user in Proxmox (e.g., `terraform@pve`)
2. Assign appropriate permissions/roles to this user (e.g., [Creating the Proxmox user and role for terraform](https://registry.terraform.io/providers/Elsvent/proxmox/latest/docs))
3. Generate an API token for the user via **Datacenter → Permissions → API Tokens**

### Disabling Privilege Separation

**Important:** When creating the API token, you must **disable "Privilege Separation"** (uncheck the box).

**Why this matters:**

Proxmox API tokens have an optional "Privilege Separation" feature that, when enabled, creates a separate permission context for the token. This means the token would need its own explicit permission assignments independent from the user account.

When privilege separation is enabled:

- The token does NOT inherit the user's permissions
- You must manually assign permissions directly to the token itself
- This adds complexity and is generally unnecessary for automation tools like Terraform

When privilege separation is disabled:

- The token inherits all permissions from the associated user account
- You only need to manage permissions in one place (the user)
- Terraform gets the same access rights as the user, ensuring it can perform all required operations

For Terraform automation, disabling privilege separation is the recommended approach because it simplifies permission management and ensures the token has sufficient privileges to create, modify, and destroy resources.

## Why Use macOS Keychain Access?

The benefits of using macOS Keychain for credentials include:

- **Encryption**: Passwords are encrypted and protected by macOS security
- **No plain text files**: Eliminates the risk of accidentally committing secrets to Git
- **Centralized management**: All credentials in one secure location
- **Access control**: Integration with macOS security policies and Touch ID/Face ID

## Storing Proxmox Credentials in your Mac's Keychain Access

To add your Proxmox API token to the Keychain, use the `security add-generic-password` command:

```bash
security add-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -D "API Token" \
  -w
```

**Parameter breakdown:**
- `-s proxmox-terraform`: The **service name** (identifier for this credential)
- `-a proxmox-terraform-user`: The **account name** (username/identifier)
- `-D API Token`: Specify kind (default is "application password"). Since we are using a token we are setting API Token
- git `-w`: Prompts you to enter the password interactively (the API token)

After running this command, you'll be prompted to enter your Proxmox API token. The token will be securely stored in your **Defualt** Keychain. In my case it is the `login` keychain. It seems there is no way to access the iCloud key chain with the security command and sync the token between Macs.

Make sure the api token use the format `<user id>!<token id>=<api token>`

**Example with the password inline** (less secure, but useful for scripting):
```bash
security add-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -D "API Token" \
  -w "your-api-token-here"
```

## Retrieving Credentials from Keychain

To retrieve the stored password, use `security find-generic-password`:

```bash
security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w
```

The `-w` flag outputs only the password (without metadata), making it perfect for scripting and automation.

**Test the retrieval:**
```bash
echo "Retrieved password: $(security find-generic-password -s proxmox-terraform -a proxmox-terraform-user -w)"
```

## Using Keychain Credentials with Terraform

Now comes the magic – automatically populating Terraform variables with credentials from Keychain.

### Terraform Variables Configuration

Create or update your `variables.tf` file:

```hcl
variable "proxmox_api_url" {
  description = "Proxmox API URL (e.g., https://proxmox.example.com:8006/api2/json)"
  type        = string
}

variable "proxmox_api_token" {
  description = "Proxmox API Toke, use environment variable or secure vault (e.g. terraform@pve!provider=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)"
  type        = string
  sensitive   = true
}

variable "proxmox_tls_insecure" {
  description = "Skip TLS verification (set to true for self-signed certificates)"
  type        = bool
  default     = false
}
```

### Provider Configuration (for bgp/proxmox)

In your `main.tf` or `provider.tf`:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = ">= 0.89.0"
    }
  }
}

provider "proxmox" {
  endpoint  = var.proxmox_api_url
  api_token = var.proxmox_api_token
  insecure  = var.proxmox_tls_insecure
}
```

### Automated Workflow with Shell Script

Create a wrapper script `terraform-proxmox.sh` to automatically retrieve the password:

```bash
#!/bin/bash

# Retrieve password from Keychain
export TF_VAR_proxmox__api_token=$(security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w)

# Check if password was retrieved
if [ -z "$TF_VAR_proxmox__api_token" ]; then
  echo "Error: Could not retrieve Proxmox password from Keychain"
  exit 1
fi

terraform "$@"
```

Make it executable:

```bash
chmod +x terraform-proxmox.sh
```

**Usage:**
```bash
./terraform-proxmox.sh init
./terraform-proxmox.sh plan
./terraform-proxmox.sh apply
```

## Complete Example Workflow

Here's a complete workflow from setup to deployment:

```bash
# 1. Store your Proxmox API token in Keychain
security add-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -D "API Token" \
  -w "your-secret-token-here"

# 2. Verify it's stored correctly
security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w

# 3. Initialize Terraform with credentials from Keychain
TF_VAR_proxmox_api_token=$(security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w) terraform init

# 4. Plan your infrastructure
TF_VAR_proxmox_api_token=$(security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w) terraform plan

# 5. Apply changes
TF_VAR_proxmox_api_token=$(security find-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -w) terraform apply
```

## Security Best Practices

1. **Never commit credentials**: Add `*.tfvars` and any credential files to `.gitignore`
2. **Use sensitive variables**: Mark password, api token or ticket xvariables as `sensitive = true` in Terraform
3. **Rotate tokens regularly**: Update your Keychain entries periodically
4. **Limit token permissions**: Create API tokens with minimal required permissions in Proxmox

## Updating Credentials

To update the stored password:

```bash
# Delete the old entry
security delete-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user

# Add the new one
security add-generic-password \
  -s proxmox-terraform \
  -a proxmox-terraform-user \
  -D "API Token" \
  -w "new-token-here"
```

## Conclusion

By leveraging macOS Keychain to store Proxmox API tokens, you can maintain a secure Terraform workflow without compromising on convenience. This approach eliminates the need for plain text credential files while keeping your infrastructure-as-code practices secure and maintainable.

The combination of Terraform and Keychain integration creates a professional workflow that's both secure and automated – perfect for homelab environments where security shouldn't be an afterthought.
