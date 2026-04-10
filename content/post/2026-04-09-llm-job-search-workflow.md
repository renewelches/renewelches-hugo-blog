---
layout: post
title: "Building an LLM-Powered Job Search Workflow with Claude and Obsidian"
subtitle: "From RAG-at-query-time to a compiled knowledge wiki that actually compounds"
date: 2026-04-09
author: "Rene Welches"
description: "How I built an Application (Tracking) System backed by Obsidian vault and Claude Code skills."
image: "/img/blog-start-banner.svg"
URL: "/2026/04/09/llm-job-search-workflow/"
tags:
  - AI
  - Claude
  - Obsidian
  - Job Search
  - Workflow
categories: [AI]
---

Job searching is, politely put, a lot of parallel state to manage. You're tracking companies and job ads — including the recurring LinkedIn postings that all start to blur together — tailoring resumes, prepping for interviews with three different people at two different firms simultaneously, and trying to remember which version of your pitch you used with which recruiter. It's a project management problem wearing a people problem disguise.

I spent some time building a workflow to handle all of this with LLM assistance — first with Claude Projects, then evolving toward something more structured after coming across [Andrej Karpathy's LLM Wiki idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). Here's how it evolved, and why I think the [Obsidian](https://obsidian.md)-plus-skills approach is genuinely better for this kind of work.

## Phase 1: Claude Desktop Projects

My starting point was a dedicated "Job-Applications" Claude Project. I uploaded my resumes, cover letter templates, and a collection of farewell notes from former colleagues.

> **Pro tip:** Those farewell notes turned out to be the most valuable thing in the vault. When Claude needs to write something that sounds like me at my best — a cover letter, a thank-you note, a LinkedIn message — it uses those notes as the tonal reference. If you have anything similar (performance reviews, commendations, messages from people who know your work), convert them to Markdown and add them to your resume pool. It's the most "human" input you can give an LLM.

For a while this worked fine. I'd paste a job description, ask for a tailored resume or cover letter, and get something reasonable back. But over time I noticed that I was managing a lot of content and knowledge outside of Claude, which led to a lot of extra manual labor.

Every query started from scratch. Ask Claude to cross-reference your resume against a JD while keeping in mind that you're also interviewing at a competitor in the same space? Which resume did I share with which company? It has to re-read everything, re-derive the connections, and synthesize the context fresh. Do it twenty times across twenty sessions, and it's doing the same rediscovery twenty times. Nothing carries forward.

## Phase 2: From Stateless Queries to a Persistent Knowledge Graph

Karpathy's LLM Wiki framing crystallized what was bothering me. The distinction he draws is between **compile-time** and **query-time** knowledge.

Claude Projects (and NotebookLM, ChatGPT file uploads, etc.) are essentially RAG systems. You attach documents; on every query the LLM re-reads the relevant chunks and synthesizes an answer from scratch. Nothing accumulates between queries.

The wiki approach flips this. When you ingest a new source, the LLM writes and updates a persistent set of interlinked markdown files — entity pages, concept pages, summaries, cross-references. The synthesis happens once at ingest time and is maintained incrementally. The wiki is a _compiled artifact_ that gets richer with every source you add. Think interpreted vs. compiled: Claude Projects interprets your documents on every query; a wiki compiles them into a persistent knowledge structure.

A few other things shift with this model:

- **The knowledge is yours on disk — and that matters.** Job search data is some of the most sensitive information you handle: salary expectations, interview feedback, recruiter names, notes about why you left your last job. It lives in a local git repo you control, not inside a cloud service's training pipeline. [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview) has different data retention policies than the standard web interface — but more importantly, a local vault means you're not uploading compensation data or confidential interview notes anywhere. For senior professionals bound by NDAs or handling sensitive material, this is a real consideration, not a footnote.
- **It compounds across sessions.** A useful insight from one interview prep session can be saved back as a new page. Your next session builds on it rather than rediscovering it.
- **You can run maintenance passes.** Ask the LLM to health-check the wiki for contradictions, stale claims, missing cross-references. That concept doesn't exist in a project-based document store.

The trade-off is real: Claude Projects has zero setup, works on mobile, and you get the full Claude experience out of the box. For a small document set, RAG is perfectly fine. But for an ongoing, multi-week job search with evolving materials — it starts to show its limits.

## Building the Vault

I decided to move everything into an Obsidian vault backed by Claude Code.

**Step one: convert everything to Markdown.** I asked Claude to convert all my project documents — multiple resume versions, cover letter drafts, the farewell notes collection, interview notes — into clean Markdown files. This took a single session and the output was immediately usable in Obsidian.

**Step two: structure the vault.** The vault lives under a `curriculum-vitae/` directory with three distinct areas: a `resume-pool/` for all canonical documents, one folder per company for active and closed applications, and a `.claude/` directory for skills. A `CLAUDE.md` at the root acts as the schema layer — it tells Claude how the vault is organized so every skill starts with the same shared understanding.

The full layout looks like this:

```
curriculum-vitae/
├── CLAUDE.md                          — schema: vault conventions, skill overview
├── .claude/
│   └── skills/
│       ├── application-tracker/
│       ├── interview-prep/
│       ├── resume-writer/
│       └── … (one folder per skill)
├── resume-pool/
│   ├── Resume - Leadership Focus.md
│   ├── Resume - Architect Focus.md
│   ├── Resume - Cloud Engineering Manager.md
│   ├── Original CV.md
│   └── Farewell Notes from Team.md
├── AcmeCorp/
│   └── VP of Platform Engineering [PHONE SCREEN]/
│       ├── JD.md
│       └── Protocol.md
├── SomeStartup/
│   └── Director of Engineering [INTERVIEWING]/
│       ├── JD.md
│       ├── Protocol.md
│       └── Interview Prep - John Smith.md
└── OtherCo/
    └── Staff Engineer [REJECTED]/
        └── Protocol.md
```

Each company gets a top-level folder. Inside it, each role gets its own subfolder whose name encodes the current application status in brackets — `[OPEN]`, `[PHONE SCREEN]`, `[INTERVIEWING]`, `[OFFER]`, `[REJECTED]`. When a status changes, the `application-tracker` skill renames the folder. The result: your entire pipeline is visible at a glance from the directory tree in Obsidian, no dashboard required.

Inside each role folder, two files do all the work:

- **`JD.md`** — a verbatim copy of the job description, saved the moment you log the opportunity. Job postings disappear; this one won't.
- **`Protocol.md`** — a running log of every event: who reached out, over which channel, what was discussed, next steps, impressions. Every interview, every email exchange, every status change gets a dated entry.

The `resume-pool/` holds multiple resume variants — a leadership-focused version, an architect-focused version, one tailored for cloud engineering management roles, and so on. When `application-tracker` logs an application, it records the specific resume that was sent using an Obsidian wiki-link in `Protocol.md`:

```markdown
**Resume Sent:** [[Resume - Architect Focus]]
```

This is more important than it sounds. A job search runs for weeks or months, and resumes evolve. You send an early version to company A, improve it for company B, and refine it again for company C. Without this link, you'll eventually lose track of what you actually submitted where. With it, clicking `[[Resume - Architect Focus]]` in Obsidian opens the exact file — and if you're in a later-stage interview, you can verify your answers align with what the hiring team has in front of them.

**Step three: create skills for every recurring task.** This is where Claude Code's skill system earns its keep. Instead of re-prompting from scratch every time I want to do something like prepare for an interview or write a networking email, I have a dedicated skill for each task that already knows the context of the vault.

I asked Claude to think through all the recurring job-search tasks I'd been doing manually and create skills for them. The core skills I rely on most:

| Skill                      | What it does                                                                                  |
| -------------------------- | --------------------------------------------------------------------------------------------- |
| `application-tracker`      | Creates and maintains per-company folders, JD copies, and a running event log — the CRM layer |
| `job-description-analysis` | Breaks down a JD — key requirements, red flags, how the role maps to my background            |
| `interview-prep`           | Prepares strategy, talking points, and story selection for a specific interview               |
| `post-interview-followup`  | Drafts thank-you notes and debrief records                                                    |
| `company-research`         | Builds a company brief — business model, tech stack, recent news, culture signals             |

Each skill lives in `.claude/skills/<skill-name>/SKILL.md` and is installed at the project level — meaning they're available whenever I open a Claude Code session in that vault directory.

**A note on the mechanical cost.** This isn't magic — it's engineered. The "compile-time" synthesis doesn't happen automatically; you trigger it. When a new JD lands in my inbox, I spend about five minutes opening the vault, running `/application-tracker`, pasting in the job description, and letting it scaffold the folder. That's the ingestion step. It's a deliberate act, not a background process, and that's fine — the friction is low enough that it becomes habit, and the payoff compounds over the length of the search.

## The CRM Layer: My Personal ATS

The `application-tracker` skill deserves its own callout because it changes what the system fundamentally _is_.

There's a (pun intended) naming collision here: ATS usually means _Applicant_ Tracking System — the software companies use to filter and rank candidates, often before a human ever reads your resume. My `application-tracker` is the opposite: an _Application_ Tracking System that works for me, not for a hiring committee.

When I log a new opportunity, the skill creates a structured folder hierarchy in the vault:

```
curriculum-vitae/
└── AcmeCorp/
    └── VP of Platform Engineering [INTERVIEWING]/
        ├── JD.md                          — verbatim copy of the job description
        ├── Protocol.md                    — running log of every event
        └── Interview Prep - Sarah Chen.md — generated by /interview-prep
```

`Protocol.md` is a CRM record for a single candidate — me — and it means I never walk into a second-round call wondering what I said in the first one.

The skill also knows when to hand off. Log a new JD? It'll offer to run `/job-description-analysis` while it's at it. Upcoming interview? "Want me to kick off interview prep?" The skills compose.

**Handling state decay.** Roles change mid-stream — a recruiter tells you they're pivoting the position from product leadership to pure platform engineering, or the job description quietly updates on their careers page. Because `JD.md` is a file you own, you can update it. When the role evolves, I paste the revised description into `JD.md`, note the change with a dated entry in `Protocol.md`, and re-run `/job-description-analysis`. The protocol entry captures the delta ("role scope narrowed, now focused on infrastructure scale-out"), so future sessions — interview prep, follow-up emails — have the accurate context rather than operating off a stale snapshot.

## Building the `application-tracker` Skill

The `application-tracker` skill didn't spring fully formed from a single prompt. I used Anthropic's [`skill-creator`](https://docs.anthropic.com/en/docs/claude-code/skills) skill — a meta-skill designed to help you author new skills — to build it.

The process was conversational. I described what I wanted: a skill that creates a company folder, saves a copy of the JD, initializes a `Protocol.md` with the intake details, and renames the folder when status changes. The `skill-creator` translated that into a structured `SKILL.md` with a clear prompt, context instructions, and output conventions. I iterated on it across a couple of sessions — adding the wiki-link for the resume sent, wiring in the offer to trigger `/job-description-analysis` on intake — until the behavior matched what I had in mind.

The takeaway: you don't have to hand-author every skill from scratch. If you can describe a recurring workflow clearly, `skill-creator` gives you a solid starting point in one session.

## A Concrete Example: Interview Prep

Say a recruiter reaches out on LinkedIn about a VP of Platform Engineering role at Acme Corp. Before anything else, I log it:

```
/application-tracker

New application:
Company: Acme Corp
Role: VP of Platform Engineering
Channel: LinkedIn recruiter outreach
Contact: Jane Doe, Senior Technical Recruiter at Acme Corp
JD: https://acmecorp.com/careers/vp-platform-engineering (fictional)
```

The skill creates `curriculum-vitae/AcmeCorp/VP of Platform Engineering [PHONE SCREEN]/` with `JD.md` and a `Protocol.md` recording the initial outreach. It then asks if I want to run a JD analysis — I do, so the role is mapped against my background while I'm still in the same session.

A week later the phone screen goes well and I have a first-round call scheduled with Sarah Chen, VP of Engineering. Back in the vault:

```
/interview-prep

Interviewer: Sarah Chen, VP of Engineering at Acme Corp
LinkedIn: https://www.linkedin.com/in/sarah-chen-acme (fictional)
```

Because `JD.md` and `Protocol.md` are already in the vault from when I logged the application, the skill can read the full context — no re-uploading, no re-explaining. It knows to:

1. **Research Sarah's background** — her title suggests an engineering leader, not a recruiter, so the prep balances technical depth with leadership narratives rather than going pure business-case.
2. **Select the right stories** from my narrative library. A VP of Engineering will want to hear about architectural authority at scale, building and growing teams, and cross-functional influence.
3. **Prepare questions to ask** calibrated to her level and the role.
4. **Flag the coding interview risk** if the role description hints at a technical screen — I have a documented weak spot there worth acknowledging.

After the call I run `/post-interview-followup`, which drafts the thank-you email and prompts a debrief. I write the debrief notes into `Protocol.md` and the folder gets renamed from `[PHONE SCREEN]` to `[INTERVIEWING]`. The full history is there, in the vault, for every subsequent session.

## What I'd Do Differently

If I were starting over, I'd convert the documents to Markdown first (it's a 20-minute job with Claude's help) and structure the vault before doing any active job searching. Working with well-organized source material from day one makes the skills dramatically more useful — they're only as good as the context they can draw on.

The one thing I'd add immediately is what `application-tracker` now gives me: log every opportunity the moment it surfaces, even if you're not sure you're interested yet. The folder costs nothing and the Protocol entry takes 30 seconds. By the time you're preparing for a second interview, you'll be very glad you recorded what happened in the first one.

The skills approach has worked well enough that I'm thinking about applying the same pattern to other ongoing projects — essentially any domain where I accumulate context over time and want an LLM collaborator that doesn't start cold every session. The vault-plus-skills model is a good primitive.

## What's Next

The vault-plus-skills model is working well, but there are two obvious gaps that I want to close.

**A job ad scanner agent.** Right now, logging a new opportunity is still a manual act — I have to notice the posting, decide it's worth tracking, and trigger `/application-tracker` myself. The next step is a scheduled Claude Code agent that monitors a job feed (an email digest or RSS feed from LinkedIn/job boards works better than scraping, which is fragile) and scores each new posting against my current target profile in the vault. Anything that hits a 6/10 match or better gets surfaced as a candidate for logging, with a brief rationale. The interesting part isn't the filtering — it's that the scoring is grounded in the vault's knowledge of what I'm actually looking for right now, not just a keyword match against a static resume.

**Email-to-Protocol auto-fill.** The biggest manual gap in the current workflow is remembering to log things. A recruiter confirms a phone screen time; I mean to note it in `Protocol.md`; I forget until the day before the call. Claude's [Gmail MCP server](https://github.com/anthropics/anthropic-quickstarts/tree/main/mcp-servers) makes a more direct solution possible: an agent that scans my inbox for job-search related emails — scheduling confirmations, status updates, rejection notices — parses the relevant details, and appends a dated entry to the right `Protocol.md` automatically. The vault already has the company/role structure; matching an email to the correct file is straightforward. This would close the loop between the real-world communication thread and the compiled knowledge record.

Both of these push the system from "LLM assistant I invoke manually" toward "stateful collaborator that keeps itself up to date." That's the direction I want to move.

## Links

- [Andrej Karpathy — LLM Wiki idea (GitHub Gist)](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Obsidian](https://obsidian.md)
- [Claude Code — Overview](https://docs.anthropic.com/en/docs/claude-code/overview)
- [Claude Code — Skills](https://docs.anthropic.com/en/docs/claude-code/skills)
