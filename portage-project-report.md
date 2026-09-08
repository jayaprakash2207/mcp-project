# Project "Portage" — A Free, Universal CLI-to-MCP Generator

*A wrapper generator that turns any command-line tool into an MCP server, without hand-coding per tool.*

---

## 1. Project Name

**Portage.**

A portage is the act of carrying a boat (or its cargo) overland between two waterways that don't otherwise connect. That's exactly what this project does: it carries an existing, non-MCP application across the gap into the MCP world, without requiring the application itself to change.

(Working name — easy to rename later; "cli2mcp-anything" or similar would also work as a plain descriptive fallback.)

---

## 2. The Problem

Claude and other AI assistants can only *act* on an application if something exposes that application's capabilities through MCP (Model Context Protocol). Most existing software was never built with MCP in mind. Today, connecting a new app to Claude means a person hand-writes a translation layer: one MCP server, one app, one round of custom coding.

That doesn't scale. There are thousands of useful command-line tools, APIs, and web apps with no MCP server, and nobody is going to hand-wrap all of them one at a time.

---

## 3. The Core Idea

Instead of wrapping *one app*, build a **generator**: a tool that looks at an app's existing, already-published interface and *automatically* produces a working MCP server for it — with no per-app code.

There are three natural interfaces a non-MCP app might already expose that a generator could read automatically:

| # | Interface the app already has | How a generator would use it |
|---|-------------------------------|-------------------------------|
| 1 | An OpenAPI/Swagger spec (common for modern SaaS APIs) | Read the spec, auto-generate one MCP tool per endpoint |
| 2 | Only a web UI, no API | Drive it with browser automation (click/type like a human) |
| 3 | Only a command-line interface | Parse its `--help` text / man page, auto-generate one MCP tool per command |

---

## 4. What the Research Found

We had a deep research pass check the current (September 2026) state of all three categories, plus free hosting options. Summary of the findings:

### 4.1 OpenAPI-to-MCP — already crowded
Multiple mature, actively maintained tools already exist:
- **harsha-iiiv/openapi-mcp-generator** (TypeScript, ~576 GitHub stars, MIT) — full-featured, handles auth, multiple transports.
- **jlowin/fastmcp** (Python, Apache-2.0) — the de facto Python framework, includes `FastMCP.from_openapi()`.
- **awslabs/openapi-mcp-server** (Python, backed by AWS, part of an ~8,500-star monorepo).

Notably, even FastMCP's own documentation warns that auto-converted OpenAPI servers perform worse for LLMs than hand-curated ones — quote from its docs: *"LLMs achieve significantly better performance with well-designed and curated MCP servers than with auto-converted OpenAPI servers."* Building another generic OpenAPI generator here would mean competing with well-resourced, mature tools for limited additional benefit.

### 4.2 Browser-automation MCP — already crowded
- **microsoft/playwright-mcp** (Apache-2.0, Microsoft-maintained) is the de facto standard: generic click/type/navigate tools driven by the accessibility tree, working on virtually any website.
- Serious competitors exist too: **Skyvern** (vision-first, handles CAPTCHA/2FA, AGPL license), **Stagehand**, **browser-use**.

Also crowded, and dominated by a well-funded incumbent (Microsoft).

### 4.3 CLI-to-MCP — the open gap
This category is genuinely underdeveloped:
- The closest existing tool, **RonieNeubauer/cli2mcp**, is version 0.1, only ~12 GitHub stars, TypeScript-only, and only supports local (stdio) connections — not remote hosting.
- The most-starred project, **njayp/ophis** (~76–87 stars), only works for Go/Cobra-based CLIs — a narrow slice of all CLI tools.
- **No tool anywhere parses man pages.**
- **Anthropic has published no official reference implementation** for CLI-to-MCP (their reference servers repo covers filesystem, git, memory, fetch, time — no generic CLI wrapper).

This is the one category where a hobbyist project could be genuinely novel rather than a clone of existing, better-resourced work.

### 4.4 Free hosting — Cloudflare Workers is the clear winner
| Host | Verdict | Key limits (as of Sept 2026) |
|---|---|---|
| **Cloudflare Workers** | **Best choice** | Free forever: 100,000 requests/day, near-zero cold start, official MCP templates + Agents SDK |
| Deno Deploy | Strong runner-up | Free: 1M requests/month, TypeScript-native, sandboxed |
| Render | Usable, but sleeps | Free, but spins down after 15 min idle → 30–60s cold start on next request |
| Railway | Not viable for $0 | Only a one-time $5 trial credit; ongoing free tier is just $1/month |
| Fly.io | Not viable for $0 | No real free tier for new signups since Oct 2024; pay-as-you-go only |

Cloudflare Workers has one important limitation for this project: it **cannot execute native binaries** (it runs in a JS/V8 sandbox), so actually *running* CLI commands can't happen there — only the MCP protocol/transport layer can.

---

## 5. The Plan — Building Portage

### Phase 1 — The Parser (core innovation)
Build a Python tool that:
- Runs `<command> --help` (and eventually reads man pages) for a given CLI tool.
- Parses the plain-text output into a structured shape: flags, positional arguments, types (string/number/boolean/enum/array), and descriptions.
- Handles nested subcommands (e.g., `git remote add`, `docker container run`).
- Converts that structure into a valid MCP tool schema (the format Claude needs to know a tool exists and how to call it).

This is the genuinely novel piece — nothing published today does both `--help` parsing *and* man-page parsing, across multiple CLI conventions (GNU/BSD/POSIX), in one general-purpose tool.

### Phase 2 — Safety Guardrails
Automatically wrapping arbitrary CLI commands is risky — a generated tool could let an AI delete files, leak secrets, or hang the system. Before anything is usable, add:
- **Allow-listing**: only expose flags/commands explicitly marked safe (or run in a strict "read-only preview" mode by default).
- **Argument validation**: reject malformed or suspicious input before it ever reaches a real shell command.
- **Timeouts**: kill any command that runs too long.
- **Sandboxed execution**: run commands in a restricted environment (limited filesystem/network access) wherever possible.

### Phase 3 — Remote Transport
Support MCP's **streamable HTTP** transport (the current standard for remote servers), not just the simple local "stdio" mode most small tools default to. This is what makes Portage usable from anywhere, not just your own machine.

### Phase 4 — Free Hosting
- Host the MCP protocol/transport layer on **Cloudflare Workers**, using Cloudflare's official free MCP server template as the starting point.
- Because Workers can't execute native CLI binaries, the actual command-execution sandbox runs elsewhere — either on **Deno Deploy's** free tier (good sandboxing via Deno's permission system) or a **Render** free web service (kept awake with periodic pings). Workers talks to that execution layer over HTTP.

### Phase 5 (stretch) — Man Page Enrichment
Once basic `--help` parsing works reliably, add man-page reading to improve tool *descriptions* — the #1 quality problem across every existing auto-generator is vague, unhelpful descriptions that confuse the AI about when to use a tool. Man pages usually have much richer explanatory text than `--help` output, so mining them is a real differentiator.

---

## 6. Fallback / Alternative Idea

If the CLI-parsing approach turns out to be more work than wanted, a simpler but still valuable alternative:

**A curated free-API marketplace** — instead of auto-generating from every possible OpenAPI spec, hand-pick a handful of good free public APIs (weather, Wikipedia, currency conversion, etc.) and build small, carefully hand-tuned MCP servers for each, with genuinely well-written tool descriptions. This directly targets the biggest known weakness of automatic OpenAPI generators (bad descriptions) with much less engineering effort than a full parser.

---

## 7. Why This Is a Good Project for a No-Budget Build

- **No paid resources required** at any stage — Python/parsing work is free, Cloudflare Workers' free tier is genuinely free forever (not a trial), and every underlying open-source tool referenced is free/MIT/Apache licensed.
- **Fills a real, verified gap** — this isn't guesswork; the research confirmed no mature or official tool currently does this.
- **Naturally extensible** — Phase 1 alone (a `--help` parser) is already a usable, shippable tool; every later phase adds value without requiring a rewrite.
- **Reusable across many apps at once** — unlike a hand-built wrapper, one working version of Portage can generate MCP servers for hundreds of different CLI tools with zero additional per-tool code.

---

## 8. Immediate Next Step

Start Phase 1: design the `--help` text parser. Concretely, that means:
1. Pick 3–5 real CLI tools to test against first (e.g., `curl`, `jq`, `git`, `ffmpeg`).
2. Write the logic that runs `<tool> --help`, captures the output, and extracts flags/arguments/descriptions using pattern-matching rules for common help-text formats.
3. Convert the extracted structure into a valid MCP tool JSON schema.
4. Test it end-to-end with a real MCP client (e.g., Claude Desktop in stdio mode) before worrying about hosting.
