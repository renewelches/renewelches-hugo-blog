---
layout: post
title: "My Local AI Stack: Mac Mini + Proxmox LXCs"
subtitle: "Ollama on the Mac mini, Open WebUI on Proxmox, and SearXNG keeping the web search private"
date: 2026-05-26 10:00:00
author: "Rene Welches"
description: "How I run a fully self-hosted AI stack split across a Mac mini M4 Pro for inference and a Proxmox cluster for the frontend — Ollama, Docling, Caddy, Open WebUI and SearXNG, all behind TLS."
image: "/img/local-ai-stack-banner.svg"
publishDate: 2026-05-26 10:00:00
tags:
  - AI
  - Ollama
  - Open WebUI
  - SearXNG
  - Docling
  - Proxmox
  - Caddy
  - Mac mini
  - Homelab
  - LXC
categories: [AI]
URL: "/2026/05/26/local-ai-stack-mac-mini-proxmox"
---

I have been slowly assembling a local AI setup that I actually want to use day to day. The goal was simple: a Claude/ChatGPT-like experience that runs on my own hardware, with web search and document RAG, and without sending my prompts off to someone else's servers. It ended up being a two-machine split — a Mac mini doing the inference, and a couple of Proxmox LXCs doing the frontend work — and I am surprisingly happy with how it landed.

This post walks through the two halves and how they talk to each other.

## Why Split It Across Two Machines?

The honest answer: my Proxmox cluster runs on mini PCs with integrated graphics and modest RAM. They are great for LXCs that mostly push HTTP around, but they are not the place I want to run a 35B-parameter language model.

The Mac mini M4 Pro, on the other hand, is genuinely good at this. Apple Silicon's unified memory means the GPU can address all 48 GB of RAM, which is exactly what local LLMs want. And since [Ollama added native MLX support](https://ollama.com/blog/mlx) in 0.19, models that ship in MLX-quantized formats run _fast_ on macOS — no compromise compared to llama.cpp on Linux for this hardware.

So the split is:

- **Mac mini** — the heavy compute (Ollama for LLMs, Docling for document parsing), all exposed over HTTPS via Caddy.
- **Proxmox LXCs** — the user-facing services (Open WebUI as the chat UI, SearXNG as the private search backend).

The frontend talks to the backend over the LAN. Everything is behind TLS, signed by my homelab CA.

## Part 1: The Mac Mini Stack

The Mac mini's job is to expose two services to the LAN:

- **Ollama** at `ollama.homelab.home`
- **Docling** at `docling.homelab.home`

Both sit behind Caddy on `:443`, with TLS certs signed by my homelab root CA (see my [homelab root CA setup post](/2026/02/18/homelab-root-ca/) if you want the full story on that).

### Ollama and the Qwen Coder Models

Ollama is the easy part. Install it, pull a model, run `ollama serve`. The only non-default thing I do is set it to listen on all interfaces so Caddy can reverse-proxy to it:

```bash
launchctl setenv OLLAMA_HOST 0.0.0.0:11434
```

For models, my current daily driver is `qwen3.6:35b-a3b-coding-mxfp8`. It's an A3B mixture-of-experts model — 35B parameters total but only ~3B active per token, so it's much faster than the size suggests, and the MXFP8 quantization plus MLX backend keeps it well within the Mac mini's memory budget.

I also tried `qwen3.6:35b-a3b-coding-nvfp4` — same model, NVFP4 quantization. Raw benchmarks were the best of any model I have run locally, but when I plugged it into [Hermes Agent](https://hermes-agent.nousresearch.com) the actual agentic results were noticeably worse than the MXFP8 build. Quantization details matter more than the headline numbers suggest, especially once you put the model in a tool-use loop where small precision differences compound. MXFP8 stayed on the dial.

### Docling for RAG

The other service running on the Mac mini is [Docling](https://github.com/DS4SD/docling) — IBM's open-source document conversion library. It takes a PDF (or DOCX, PPTX, HTML, image-with-OCR, etc.) and returns clean Markdown or structured JSON. It runs as `docling-serve` exposing a REST API on `:5001`.

The reason this lives on the Mac mini and not in a Proxmox LXC is the same reason Ollama does: Docling uses ML models for layout detection, table extraction, and OCR, and it runs much faster with Apple's MPS GPU backend than on a small LXC CPU. The launchd plist sets `PYTORCH_ENABLE_MPS_FALLBACK=1` so any unsupported ops gracefully fall back to CPU, and a fresh document conversion takes a couple of seconds instead of a couple of minutes.

The first request after a service start is slow — Docling lazy-loads its ML models — but everything after that is snappy.

### Caddy as the TLS Front Door

Caddy is the glue. One Caddyfile, two reverse-proxy blocks, TLS terminated using certs from my homelab CA:

```caddy
{
    auto_https off
    admin localhost:2019
}

ollama.homelab.home {
    tls /Users/rene/.../mac-mini.crt /Users/rene/.../mac-mini.key

    reverse_proxy localhost:11434 {
        transport http {
            read_timeout 300s
            write_timeout 300s
        }
    }
}

docling.homelab.home {
    tls /Users/rene/.../mac-mini.crt /Users/rene/.../mac-mini.key

    reverse_proxy localhost:5001 {
        transport http {
            read_timeout 600s
            write_timeout 600s
        }
    }
}
```

Two things worth calling out:

1. **Long timeouts.** Ollama responses can run for minutes if the model is thinking or streaming a long generation. Docling can take a similarly long time on a big PDF. Default Caddy timeouts will cut you off mid-response. I had to learn this twice before it stuck.
2. **`auto_https off`.** Caddy normally tries to get certs from Let's Encrypt automatically. Since these hostnames only resolve inside my LAN (via AdGuard DNS rewrites) and use my homelab CA, I disable that behavior and point Caddy at the local cert files explicitly.

DNS is handled by AdGuard Home running in my homelab — two rewrites point `ollama.homelab.home` and `docling.homelab.home` at the Mac mini's IP. Both Caddy and `docling-serve` are managed by launchd so they survive reboots and the inevitable macOS update cycle.

## Part 2: The Proxmox LXC Stack

The frontend half runs on my Proxmox cluster as two independent LXC containers, both deployed with the Terraform + Ansible setup I [wrote about earlier](/2026/04/20/proxmox-lxc-iac/). The stack is called `openwebui-searxng` and lives in [`proxmox-lxc-iac`](https://github.com/renewelches/proxmox-lxc-iac).

### Open WebUI

[Open WebUI](https://github.com/open-webui/open-webui) is the UI layer. It looks and feels like ChatGPT, supports multiple model backends, has built-in RAG, web search, conversation history, model parameter tweaking — basically everything you would want from a self-hosted chat frontend.

It runs as a Docker container inside the LXC, behind an Nginx sidecar that handles TLS. The Ansible playbook configures the relevant pieces via the container's environment:

```bash
OLLAMA_BASE_URL=https://ollama.homelab.home
ENABLE_WEB_SEARCH=True
WEB_SEARCH_ENGINE=searxng
SEARXNG_QUERY_URL=https://<searxng-lxc-ip>/search?q=<query>
WEB_SEARCH_RESULT_COUNT=4
WEB_SEARCH_CONCURRENT_REQUESTS=10
```

That `OLLAMA_BASE_URL` is the whole point of this architecture: Open WebUI does not run Ollama locally inside the LXC — it talks to the Mac mini over the LAN. From Open WebUI's perspective, the Mac mini is just an Ollama server. From the Mac mini's perspective, Open WebUI is just another HTTP client. Clean separation, and I can swap either side without touching the other.

For RAG, Open WebUI has a "Content Extraction Engine" setting that accepts an external Docling endpoint. Point it at `https://docling.homelab.home`, drop a PDF into a chat, and Docling on the Mac mini converts it to Markdown that Open WebUI then chunks and feeds into the prompt. Documents in, useful answers out, all of it staying on my LAN.

### SearXNG for Private Web Search

[SearXNG](https://github.com/searxng/searxng) is a meta-search engine — it queries other search engines on your behalf and aggregates the results without tracking you or building a profile. For Open WebUI's web search feature, this is exactly what you want: the LLM gets fresh web context, but no search provider gets a log of your prompts.

It runs in its own LXC, also as Docker behind Nginx, with its own TLS cert. The Ansible playbook drops a custom `settings.yml` into the container so I can tune which search engines it queries and a few rate limits.

The thing that bit me on first setup: Open WebUI talks to SearXNG over HTTPS using my homelab CA. If the CA is not trusted inside the Open WebUI container, the TLS handshake fails silently and web search just stops working with no useful error in the UI. The Ansible playbook mounts the host's CA bundle into the container at `/etc/ssl/certs/ca-certificates.crt`, and Open WebUI's environment points `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE` at it. With that in place, web search works.

## Putting It All Together

Here is the full flow when I ask Open WebUI to "summarize the attached PDF and find related recent news":

```
                    ┌───────────────────────────────────────┐
                    │ Mac mini (192.168.1.36)              │
                    │                                       │
        inference   │  ┌─────────────┐                      │
   ┌──────────────▶ │  │ Ollama      │  qwen3.6:35b-...     │
   │                │  │ :11434      │                      │
   │                │  └─────────────┘                      │
   │  document      │                                       │
   │   parsing      │  ┌─────────────┐                      │
   │  ┌───────────▶ │  │ Docling     │                      │
   │  │             │  │ :5001       │                      │
   │  │             │  └─────────────┘                      │
   │  │             │                                       │
   │  │             │  Caddy :443 reverse-proxies both      │
   │  │             │  behind homelab-CA TLS                │
   │  │             └───────────────────────────────────────┘
   │  │
   │  │ HTTPS (LAN)
   │  │
┌──┴──┴────────────────────┐         ┌─────────────────────────┐
│ Open WebUI LXC           │ HTTPS   │ SearXNG LXC             │
│ (Proxmox cluster)        │ ──────▶ │ (Proxmox cluster)       │
│                          │         │                         │
│ Browser ──▶ :443 (Nginx) │         │ Meta-search → DuckDuckGo│
│            ──▶ Open WebUI│         │              Brave, etc │
└──────────────────────────┘         └─────────────────────────┘
```

- The browser hits Open WebUI's Nginx over HTTPS.
- Open WebUI calls Ollama on the Mac mini for the model response.
- For the PDF: Open WebUI sends it to Docling, gets Markdown back, chunks it into the prompt.
- For the "recent news" part: Open WebUI calls SearXNG, which fans out to public search engines and returns aggregated results, which Open WebUI then folds into the prompt.
- The model on the Mac mini generates the answer using all of that context.

Everything is encrypted in transit. Nothing leaves my LAN except the anonymized search queries SearXNG sends out — which is the same thing my browser would do anyway.

## What Works, What's Next

What I like about this setup:

- **It's not a single container of doom.** Each piece is independent. I can update Ollama without touching Open WebUI. I can re-provision the SearXNG LXC without losing chat history. The Mac mini does the heavy compute; the cluster does the wrappers around it.
- **Native MLX on Ollama makes a real difference.** I was on llama.cpp via Ollama for a while; switching to MLX-format Qwen models was a meaningful speedup on Apple Silicon and freed up enough memory headroom to keep more in cache.
- **TLS everywhere paid off.** It was a hassle to set up — and the SearXNG/CA-bundle gotcha cost me an hour — but everything is consistent now: same root CA, same trust store, same expectations across the homelab.

What's next on the list:

- Replacing the four shell scripts that currently provision the Mac mini with a proper Ansible setup and putting it on GitHub. The scripts work, but they live in the wrong shape — Ansible is what the rest of the homelab uses, and the Mac mini should not be the odd one out.
- Trying out Open WebUI's "Tools" feature to wire up local file access and a few homelab APIs.
- Experimenting with the new Qwen3.6 builds as they get re-quantized — the gap between MXFP8 and NVFP4 in agentic workloads is interesting and worth more poking.
- Eventually adding a small model on the Proxmox side too, for fast embedding-only workloads that don't need to wake up the Mac mini.
