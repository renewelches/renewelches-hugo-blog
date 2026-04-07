---
layout: post
title: "Switching from Black to Ruff for Python Formatting in VS Code"
subtitle: "One tool to replace Black, isort, and flake8 — faster and less strict"
date: 2026-04-07
author: "Rene Welches"
description: "Why I replaced Black and isort with Ruff in VS Code, how the new configuration looks, and what it improves — especially when following along with Anthropic and other third-party code examples."
image: "/img/2026-04-07-ruff-vscode-banner.svg"
URL: "/2026/04/07/vscode-python-formatting-ruff/"
tags:
  - VS Code
  - python
  - ruff
categories: [Development]
---

# Switching from Black to Ruff for Python Formatting in VS Code

In a [previous post](/2025/11/11/vscode-python-formatter/) I set up Black, isort, and Mypy to keep my Python code clean in VS Code. That setup worked well for a while, but I kept running into a frustrating problem: whenever I followed along with official examples from Anthropic or other third-party libraries, VS Code would light up with dozens of errors. The strict Black + Mypy configuration was fighting me more than helping me.

Time for an upgrade.

## What's Wrong with the Old Setup?

A few things started to bother me:

**Black + isort is two tools doing one job.** Black handles formatting and isort handles imports, but they need to be kept in sync (hence `--profile black` in the isort config). If they disagree, you get conflicts.

**`python.analysis.typeCheckingMode: strict` is very aggressive.** Strict mode requires full type annotations everywhere — including in code you didn't write. When pasting an example from the Anthropic SDK docs or a tutorial, you'd immediately get a wall of red squiggles for missing type hints. It was driving me crazy!

**`--disallow-untyped-defs` in Mypy compounds the problem.** This flag means Mypy errors on every function that doesn't have type annotations. Again, fine for a fully-typed production codebase, but painful when experimenting or learning.

## Enter Ruff

[Ruff](https://docs.astral.sh/ruff/) is a Python linter and formatter written in Rust. It replaces Black, isort, pyflakes, pycodestyle, and more — all in a single tool that runs 10–100x faster. It's become the community standard quickly, and for good reason:

- **One extension instead of two** — no more keeping Black and isort in sync
- **Same formatting output as Black** — if you liked Black's style, Ruff's formatter produces identical results
- **Built-in import sorting** — the `I` ruleset replaces isort entirely
- **Sensible defaults** — the community-standard rule set catches real problems without being pedantic

## The New VS Code Configuration

Here's the relevant part of `settings.json` after the switch:

```json
{
  "[python]": {
    "editor.formatOnSave": true,
    // Ruff replaces Black as the formatter (extension: charliermarsh.ruff)
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.codeActionsOnSave": {
      // Ruff handles import sorting — no isort extension needed
      "source.organizeImports": "explicit",
      // Auto-fix lint violations on save
      "source.fixAll": "explicit"
    },
    "editor.wordBasedSuggestions": "off"
  },

  // "basic" catches real type errors without requiring full annotations everywhere
  "python.analysis.typeCheckingMode": "basic",

  // Community-standard lint rules:
  //   E = pycodestyle errors, F = pyflakes, I = isort, W = pycodestyle warnings
  // E501 (line too long) is ignored because Ruff's formatter handles line length
  "ruff.lint.select": ["E", "F", "I", "W"],
  "ruff.lint.ignore": ["E501"],

  "mypy-type-checker.enabled": true,
  "mypy-type-checker.args": [
    "--ignore-missing-imports", // No errors for third-party packages without stubs
    "--follow-imports=silent", // Only check our own code, not imported packages
    "--show-column-numbers"
  ],
  "mypy-type-checker.reportingScope": "workspace"
}
```

## What Changed and Why It's Better

|                             | Old setup                             | New setup                                     |
| --------------------------- | ------------------------------------- | --------------------------------------------- |
| **Formatter**               | `ms-python.black-formatter`           | `charliermarsh.ruff`                          |
| **Import sorting**          | `ms-python.isort` + `--profile black` | Built into Ruff (`I` ruleset)                 |
| **Type checking level**     | `strict`                              | `basic`                                       |
| **Mypy untyped functions**  | Error (`--disallow-untyped-defs`)     | Allowed                                       |
| **Missing package stubs**   | Error                                 | Silently ignored (`--ignore-missing-imports`) |
| **Line length enforcement** | Pycodestyle E501                      | Handled by formatter, not linter              |

### Fewer false alarms on third-party examples

Switching type checking from `strict` to `basic` was the single biggest quality-of-life improvement. Pylance's `basic` mode still catches real mistakes — using a variable before assignment, passing the wrong type, calling a method that doesn't exist — but it doesn't demand that every parameter and return value be annotated. When copying an Anthropic SDK example or a snippet from a tutorial, VS Code now stays quiet unless something is genuinely wrong.

The same logic applies to the Mypy changes. Dropping `--disallow-untyped-defs` and adding `--ignore-missing-imports` means Mypy focuses on actual type errors in your own code, rather than complaining about unannotated functions or missing stubs for packages like `anthropic`, `boto3`, or `requests`.

### One fewer extension to manage

Removing the isort extension simplifies the setup. Ruff's `I` ruleset implements the same sorting logic, so you get the same organized imports on save without a separate tool that needs to be kept in sync with the formatter.

## Installation

Install the Ruff extension from the VS Code marketplace:

- [Ruff](https://marketplace.visualstudio.com/items?itemName=charliermarsh.ruff)

You can uninstall the Black Formatter and isort extensions — Ruff covers both. Keep the Mypy Type Checker extension if you want the additional static analysis layer on top of Pylance.

Optionally install Ruff into your Python environment too (useful for running it in CI or pre-commit hooks):

```bash
pip install ruff
```

That's it. The experience is noticeably snappier, the noise level is much lower, and the formatting output is identical to what Black produced.
