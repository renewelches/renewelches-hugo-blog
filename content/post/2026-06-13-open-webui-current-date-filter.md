---
layout: post
title: "Back to the Future: Bringing Ollama Date-Blind Models to the Present"
subtitle: "My local Ollama model was stuck in 2024 and treated the real present as a fake future — here's the Open WebUI filter that gets it current"
date: 2026-06-13 10:00:00
author: "Rene Welches"
description: "A local LLM stuck at its 2024 training cutoff accused me of faking the future when web search returned same-day results. Here is why it happens in Open WebUI and a small filter function that drives any model back to the present."
image: "/img/local-ai-stack-banner.svg"
publishDate: 2026-06-13 10:00:00
tags:
  - AI
  - Open WebUI
  - Ollama
  - Gemma
  - Qwen
  - LLM
  - Self-hosting
  - Homelab
categories: [AI]
URL: "/2026/06/13/open-webui-current-date-filter"
---

A while back I wrote about [my local AI stack](/2026/05/26/local-ai-stack-mac-mini-proxmox/) — Ollama on a Mac mini, Open WebUI on a Proxmox LXC, SearXNG keeping web search private. It mostly just works. Then one day I noticed something strange.

I asked Gemma 4 to research a topic with web search — something that had just been published that same day in a blog post. The search results came back, full of 2026 dates, and instead of using them Gemma 4 pushed back: it accused me of _testing_ it, as if I'd planted fake future-dated content to see whether it would fall for it. As far as Gemma was concerned it was still some day in 2024, so the actual present looked like a setup. Meanwhile Ministral, running on the same Ollama instance through the same Open WebUI, knew exactly what day it was.

That asymmetry is the whole story, so let's pull it apart.

## Why a model "knows" the date at all

A language model has no clock. Its weights were frozen at some training cutoff, and left to its own devices it will assume the present is roughly whenever its training data ended. The only way it knows today's date is if something **tells** it — usually a line in the system prompt like _"The current date is Saturday, June 13, 2026."_

So the real question isn't "why doesn't Gemma know the date," it's "who is responsible for injecting it?" And it turns out the answer differs depending on the model _and_ the frontend.

Take a look at Ministral's Modelfile:

```bash
ollama show ministral-3:14b --modelfile | head -20
```

```
TEMPLATE """...
[SYSTEM_PROMPT]You are Ministral-3-14B-Instruct-2512...
Your knowledge base was last updated on 2023-10-01.
The current date is {{ currentDate }}.
...
You are always very attentive to dates, in particular you try to resolve
dates (e.g. "yesterday" is {{ yesterdayDate }})...
```

Mistral baked the date directly into the chat template. `{{ currentDate }}` is an Ollama template function that gets filled in at inference time, so **every** request through Ministral carries the correct date, no matter what frontend you use. That's why it always worked for me.

Now Gemma 4:

```bash
ollama show gemma4:26b-mxfp8 --modelfile | head -20
```

```
TEMPLATE {{ .Prompt }}
RENDERER gemma4
PARSER gemma4
PARAMETER temperature 1
...
```

No date anywhere. Gemma 4 uses Ollama's newer `RENDERER` / `PARSER` mechanism, where the chat formatting is handled by built-in Go code rather than the template, and that template is just `{{ .Prompt }}`. Nothing injects a date. So unless the _frontend_ supplies one, Gemma is flying blind.

This is also why it behaved differently across my tools. When I run Gemma through [Hermes Agent](https://hermes-agent.nousresearch.com), Hermes injects the date into the prompt itself, so Gemma is fine. When I run the exact same model through Open WebUI, nothing injects it — and Gemma reverts to believing it's 2024.

So the bug isn't Gemma's. It's a gap in responsibility: some models self-inject the date, some frontends inject it, and when neither does, the model is left guessing.

## Why I didn't just fix the Modelfile

The obvious fix is to do what Mistral did — edit Gemma's Modelfile and bake the date into the template. I decided against it for two reasons:

1. **Gemma's rendering is in Go, not the template.** With `RENDERER gemma4` doing the formatting, hand-editing `TEMPLATE {{ .Prompt }}` to splice in a system block is fragile and easy to get subtly wrong.
2. **It doesn't survive updates.** Every `ollama pull` would blow my edit away, and I'd be maintaining a custom Modelfile per model forever.

I'd rather fix this once, at the layer where _most of_ my chat traffic already flows: Open WebUI.

## Option A: the one-line native fix

Before reaching for code, it's worth knowing Open WebUI has this built in. It supports system-prompt variables that get substituted at request time:

- `{{CURRENT_DATE}}`
- `{{CURRENT_DATETIME}}`
- `{{CURRENT_TIME}}`
- `{{CURRENT_TIMEZONE}}`
- `{{CURRENT_WEEKDAY}}`

Drop `Today is {{CURRENT_WEEKDAY}}, {{CURRENT_DATE}}.` into a model's **System Prompt** field (Workspace → Models → edit the model) and you're done. No code.

If you only have one misbehaving model **and that model has a system role**, do this and stop reading. It's the least-effort fix that exists.

The reasons I didn't stop here:

- It's **per-model and manual** — I'd have to set it on Gemma, on Qwen, and on the next model I pull that has the same problem, remembering each time.
- It only lives in the **system-prompt field** — which, as I found out the hard way below, is useless for models like Gemma 1/2/3 that have no system role. The variable gets substituted, then the whole system message gets dropped.
- The injected date has historically been **server/UTC time**, not the user's local time ([open-webui#9754](https://github.com/open-webui/open-webui/issues/9754)), and there have been rendering-context bugs ([#16019](https://github.com/open-webui/open-webui/issues/16019)).

I wanted one thing I could write once, control fully, and switch on for any model that needs it. That's a filter.

## Option B: a Filter function

Open WebUI's [Filter functions](https://docs.openwebui.com/features/extensibility/plugin/functions/filter/) let you hook the request pipeline. The `inlet` method runs **before** the request hits the model, so it's the natural place to inject context. Here's mine:

```python
"""
title: datetime filter
author: rene welches
version: 0.5
"""

import datetime
import re
from typing import Optional
from pydantic import BaseModel, Field


class Filter:
    class Valves(BaseModel):
        priority: int = Field(
            default=0, description="Priority level for the filter."
        )
        date_format: str = Field(
            default="%A, %B %d, %Y",
            description="strftime format for the injected date.",
        )

    def __init__(self):
        self.valves = self.Valves()

    def inlet(self, body: dict, __user__: Optional[dict] = None) -> dict:
        messages = body.get("messages", [])
        if not messages:
            return body

        # Build the fresh date notice.
        try:
            current_date = datetime.datetime.now().strftime(
                self.valves.date_format
            )
        except Exception as e:
            print(f"datetime filter: bad date_format ({e}), using default")
            current_date = datetime.datetime.now().strftime("%A, %B %d, %Y")

        notice = (
            f"[Current date: {current_date}. "
            f"Trust this date for any temporal reasoning.]"
        )
        notice_pattern = r"\[Current date:.*?\]\s*"

        # Inject into the LAST user turn, not a system message. Every model
        # supports the user role, so the date reaches models with no system
        # role (Gemma 1/2/3) and models that have one (Gemma 4) alike. Strip
        # any prior notice first so re-runs don't stack.
        for msg in reversed(messages):
            if msg.get("role") == "user":
                cleaned = re.sub(notice_pattern, "", msg.get("content", ""))
                msg["content"] = f"{notice}\n\n{cleaned}"
                return body

        return body
```

What it does, in order:

- Formats today's date (configurable via the `date_format` valve, with a safe fallback if you typo the format string).
- Builds a short `[Current date: ...]` line that explicitly tells the model to _trust_ this date — which is the part that stops the model from arguing with the search results.
- **Prepends it to the most recent user message** (not a system message — see below for the hard-won reason), stripping any prior notice first so re-running the filter never stacks duplicates.

### Why the user turn, and not the system prompt

My first version injected a `system` message instead. That's the obvious choice, and it worked fine on Gemma 4 and Qwen. Then I tested it on the smallest Gemma I had, `gemma3:1b-it-q8_0`: filter switched on, system prompt left empty, and I asked it the date. It answered:

> Today is Sunday, June 14, 2024.

Wrong year, wrong day — the injected date never reached the model. And that tracks with the architecture. **Gemma 1, 2, and 3 have no system role at all** — they only understand `user` and `model` turns ([Google's prompt docs spell this out](https://ai.google.dev/gemma/docs/core/prompt-structure)). I'd assumed Ollama would fold a stray system message into the first user turn the way Hugging Face's template does; for `gemma3` here it plainly didn't, and the system content was dropped on the floor. The only reason the system-message version worked on Gemma 4 is that **Gemma 4 added native `system`-role support** ([model card](https://ai.google.dev/gemma/docs/core/model_card_4), [ollama.com/library/gemma4](https://ollama.com/library/gemma4)) — there was finally a system block for the date to land in.

The portable fix is to stop depending on the system role entirely and prepend the date to the latest **user** message. Every model understands the user role, so the date lands in front of Gemma 3 and Gemma 4 the same way — no special-casing per model. That's what the code above does, and it's the reason it survived contact with a 1B model that has no system role to speak of.

### Installing it

1. In Open WebUI go to **your user menu (bottom left) → Admin Panel → Functions → +** (the plus to create a new function).
2. Paste the code, give it a name, save.
3. Enable it. You can switch it on **globally** under **Admin Panel → Functions → [...] on your function → Global toggle**, or **per-model** by editing a model and adding the filter to its Filters list. I enabled it per-model for `gemma4` and `qwen3.6`, since Ministral already handles its own date.

That's it. No restart, no Modelfile surgery.

## Did it work?

Yes. Gemma 4 now opens every conversation already knowing the date, so when SearXNG hands it a pile of 2026 results, it treats them as the present instead of as someone trying to gaslight it. Qwen 3.6 behaves too. And the model that forced the user-turn rewrite — `gemma3:1b`, which shrugged off the system-message version and insisted it was June 2024 — now answers with the correct date once the filter writes it into the user turn. That last one is the real proof: if it lands on a 1B model with no system role, it lands everywhere.

Because the filter lives at the Open WebUI layer, the next model I pull with the same blind spot is one toggle away from being fixed — no per-model prompt editing, no custom Modelfiles to babysit.

The broader lesson I keep relearning with local models: a lot of "the model is dumb" moments are really "nobody gave the model the context a hosted API would have injected for free." The date is the simplest possible example. Worth knowing where, in your own stack, that context is supposed to come from — and noticing when nothing is supplying it.
