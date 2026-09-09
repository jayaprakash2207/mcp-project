# Project Report: Portage
### A Free, Open-Source CLI-to-MCP Generator

**Prepared by:** Jayaprakash A R
**Date:** September 2026
**Status:** Pre-development (planning complete, build not yet started)
**License (planned):** MIT or Apache-2.0 (permissive, to encourage adoption)

---

## 1. Executive Summary

**Portage** is a free, open-source tool that automatically converts any command-line interface (CLI) tool into a Model Context Protocol (MCP) server — without anyone hand-writing per-tool code. It works by parsing a CLI's own `--help` output and man pages, inferring the tool's structure (commands, flags, arguments, types), and generating a working MCP server that exposes each CLI command as a callable tool for AI assistants like Claude.

The project exists because of a specific, verified gap: as of September 2026, the two "easy" categories of automated MCP generation — OpenAPI-to-MCP and browser-automation MCP — are already dominated by mature, well-funded projects (Microsoft's Playwright MCP, AWS Labs' OpenAPI generator, FastMCP, etc.). The CLI-to-MCP category, by contrast, has no project above roughly 100 GitHub stars, no man-page parsing capability anywhere, and no official reference implementation from Anthropic or any major vendor. Portage is designed to fill that gap.

The entire project is scoped to be buildable and hostable at **zero cost**, using free tiers of developer tools and cloud platforms.

---

## 2. Problem Statement

MCP has become the standard way for AI assistants to interact with external tools and applications. But the number of MCP servers that exist is a small fraction of the number of applications people actually use. For an app to be usable by an AI assistant via MCP, someone has to write an MCP server for it — a translation layer that exposes the app's real functionality as MCP "tools."

This is especially true for **command-line tools** — the thousands of CLIs developers, sysadmins, and power users already rely on (`ffmpeg`, `imagemagick`, `curl`, `git`, custom internal scripts, etc.). Almost none of them have an MCP server, and hand-writing one for each CLI is repetitive, tedious work: every CLI already documents its own interface via `--help` and man pages, but nobody has built a robust, general tool to read that documentation and generate the wrapper automatically.

**The specific problems Portage solves:**
- No generic tool converts an arbitrary CLI's `--help` output into a validated, typed MCP tool schema.
- No tool parses man pages for this purpose at all.
- Existing attempts are early-stage, narrow (e.g., Go/Cobra-only), or manual (requiring hand-written YAML config per tool).
- Wrapping arbitrary CLI commands safely (without giving an AI assistant the ability to run dangerous commands unchecked) is an unsolved design problem in the existing small projects.

---

## 3. Research Findings (Summary)

Before committing to this direction, research was conducted across three possible approaches to "auto-generating MCP servers for non-MCP apps":

| Approach | State of the ecosystem (Sept 2026) | Verdict |
|---|---|---|
| **OpenAPI/Swagger → MCP** | Mature and crowded. Notable projects: `harsha-iiiv/openapi-mcp-generator` (~576 stars), `jlowin/fastmcp` (Python, backed by the FastMCP framework), `awslabs/openapi-mcp-server` (AWS-backed, ~8,500 stars in parent repo). Even these tools' own creators warn that auto-converted API wrappers perform worse for AI agents than hand-curated ones. | Too crowded to differentiate in. |
| **Browser automation → MCP** | Mature and crowded. Microsoft's `playwright-mcp` is the de facto standard (accessibility-tree based, works on any site). Serious competitors: Skyvern (vision-based, handles CAPTCHA/2FA), Stagehand, browser-use. | Too crowded; dominated by Microsoft. |
| **CLI (`--help`/man pages) → MCP** | Wide open. Best existing attempt (`RonieNeubauer/cli2mcp`) is version 0.1, ~12 stars, TypeScript-only, stdio-only, no man-page support. Next closest (`njayp/ophis`, ~76–87 stars) only works on Go/Cobra CLIs specifically, not general `--help` parsing. No Anthropic reference implementation exists for this category. | **Genuine, verified gap. This is Portage's chosen lane.** |

**Free hosting research** identified **Cloudflare Workers** as the best free hosting platform for the MCP-facing layer: a genuinely permanent free tier (100,000 requests/day, near-instant cold starts, official MCP server templates and an "Agents SDK"), significantly better than Render (free tier sleeps after 15 minutes, causing 30–60 second cold starts), Railway and Fly.io (no meaningful permanent free tier as of 2026), leaving Deno Deploy as a solid TypeScript-native runner-up.

One key architectural constraint surfaced by this research: Cloudflare Workers cannot execute native binaries (it runs in a V8 isolate sandbox), so the part of Portage that actually *runs* CLI commands cannot live entirely on Workers — this shapes the architecture described in Section 5.

---

## 4. Project Name & Identity

**Name:** Portage

**Why this name:** "Portage" refers to the act of carrying a boat (or its cargo) overland between two bodies of water — a historical term for bridging two things that don't otherwise connect. It's a fitting metaphor for a tool that carries CLI functionality across into the MCP "waterway" so AI assistants can use it.

**Tagline (draft):** *"Every CLI, one command away from your AI assistant."*

**One-line pitch:** Portage automatically turns any command-line tool into an MCP server by reading the tool's own `--help` and man page documentation — no hand-coding, no per-tool wrappers.

---

## 5. Product Vision & Scope

### 5.1 What Portage does (MVP)
1. Takes the name of an installed CLI tool (e.g., `ffmpeg`, `git`, `curl`, or a user's own custom script).
2. Runs `<tool> --help` (and, where available, reads the man page) to capture its documentation text.
3. Parses that text to identify:
   - Available subcommands (e.g., `git remote add`, `git commit`)
   - Flags and their types (boolean switches, string values, numeric values, enums, repeatable flags)
   - Positional arguments
   - Human-readable descriptions for each, to make the generated tool easier for an AI to use correctly
4. Generates a valid MCP tool schema (JSON Schema-based) for each command.
5. Registers these tools on an MCP server that a client (like Claude) can connect to and call.
6. Executes the actual CLI command on the user's behalf when a tool is called, returning the output.

### 5.2 What Portage explicitly does NOT do (out of scope for MVP)
- It does not attempt to wrap GUI-only applications (that's the browser-automation lane, already well served elsewhere).
- It does not attempt to guess undocumented behavior — if a CLI's `--help` is unclear or absent, Portage should fail gracefully rather than guess.
- It does not provide a hosted, multi-tenant SaaS in the MVP — the initial version is self-hosted / run-your-own-instance.

### 5.3 Safety design (critical — must not be skipped)
Because Portage gives an AI assistant the ability to execute real commands on a real machine, safety cannot be an afterthought:
- **Allow-listing:** by default, only explicitly approved commands/flags can be executed; nothing runs unless the user has reviewed and approved the generated tool definition.
- **Argument validation:** every argument is validated against its inferred type/schema before execution — no raw string concatenation into a shell command (to prevent injection attacks).
- **Timeouts:** every command execution has a maximum run time, after which it's killed.
- **Sandboxing (stretch goal):** run commands inside a restricted environment (e.g., a container or restricted user account) rather than directly on the host.
- **Dry-run mode:** an option to preview exactly what command would be executed, without running it, useful for trust-building during early use.

---

## 6. Technical Architecture

### 6.1 High-level flow

```
AI Assistant (e.g., Claude)
        │  (MCP protocol, streamable HTTP or stdio)
        ▼
Portage MCP Server (protocol + tool registry layer)
        │  (internal call: "run this validated command")
        ▼
Portage Execution Engine (sandboxed command runner)
        │  (shells out to)
        ▼
The actual CLI tool (ffmpeg, git, curl, custom script, etc.)
```

### 6.2 Two-part split (because of the Cloudflare Workers constraint)
Since Cloudflare Workers cannot run native binaries, Portage is split into two cooperating components:

1. **Portage-Protocol** (the MCP-facing layer)
   - Handles MCP handshake, tool discovery, and request/response formatting.
   - Can run on Cloudflare Workers' free tier for the "talking to Claude" side of things.
   - Talks to the Execution Engine over an internal API call.

2. **Portage-Engine** (the command execution layer)
   - Where `--help`/man-page parsing and actual command execution happen.
   - Needs a runtime that can execute real subprocesses — this cannot run on Workers.
   - Candidates: Deno Deploy (permission-sandboxed, TypeScript-native) or a free-tier Render service kept alive with a periodic ping.
   - For a purely local/personal setup (simplest MVP), this can just run on the user's own machine via `stdio` transport, skipping remote hosting entirely at first.

### 6.3 Suggested initial tech stack
- **Language:** Python (broadest ecosystem for CLI tooling, good text-parsing libraries, and the official MCP Python SDK is mature).
- **MCP SDK:** official `mcp` Python package.
- **Parsing:** custom parser for `--help` text (regex + heuristics for GNU/BSD/POSIX conventions) plus a man-page reader (using `man` command output or the `groff`/`man-db` text output).
- **Schema generation:** JSON Schema, following MCP's tool definition format.
- **Transport:** support both `stdio` (for local/personal use) and streamable HTTP (for remote hosting later).
- **Execution sandboxing:** Python's `subprocess` module with strict argument lists (never shell=True with raw strings), timeouts via `subprocess.run(..., timeout=N)`.

### 6.4 Hosting plan (once ready to go remote)
- **Portage-Protocol** → Cloudflare Workers free tier, using Cloudflare's official remote-MCP server template as a starting point.
- **Portage-Engine** → Deno Deploy free tier (preferred, TypeScript-native, sandboxed via Deno's permission system) or a keep-alive'd Render free web service.
- **State/config storage** (if needed later, e.g., saved allow-lists per user) → Cloudflare KV or D1 (both included in the free tier).

---

## 7. Development Roadmap

### Phase 0 — Setup (before writing product code)
- Set up GitHub repo, license, README, contribution guidelines.
- Set up local Python dev environment.

### Phase 1 — Core parser (MVP, local-only)
- Build the `--help` output parser for common conventions (GNU-style long/short flags, positional args, enums, repeatable flags).
- Test against a curated set of real-world CLIs: `jq`, `ripgrep`, `curl`, `git` (with subcommands), `ffmpeg`.
- Generate valid MCP tool schemas from parsed output.
- Run as a local `stdio` MCP server — connect it to Claude Desktop or Claude Code locally and confirm tools are callable.

### Phase 2 — Safety layer
- Add allow-listing config (which commands/flags are permitted).
- Add strict argument validation before execution.
- Add timeouts and basic sandboxing.
- Add a dry-run mode.

### Phase 3 — Man-page support
- Extend the parser to also read man pages (via `man <tool>` piped output), improving descriptions and catching commands whose `--help` is sparse.
- Handle nested subcommands better (e.g., multi-level command trees like `git remote add`).

### Phase 4 — Remote hosting
- Split into Portage-Protocol and Portage-Engine.
- Deploy Portage-Protocol to Cloudflare Workers using their official MCP template.
- Deploy Portage-Engine to Deno Deploy or Render.
- Add streamable HTTP transport support.

### Phase 5 — Polish & release
- Documentation, example configs, a short demo video/GIF.
- Publish to GitHub, submit to the official MCP server registry / community lists (e.g., PulseMCP, the modelcontextprotocol/servers community list).
- Announce publicly (e.g., the planned LinkedIn post).

---

## 8. Risks & Open Questions

- **Security risk is the biggest concern.** Wrapping arbitrary CLI tools for AI execution is inherently risky; the allow-list and validation design must be solid before any public release, and the MVP should default to the most restrictive settings.
- **Parsing reliability.** `--help` output formatting varies a lot between tools; the parser needs sensible fallback behavior (e.g., a generic "pass raw args" mode) when it can't confidently parse a tool's structure.
- **Competition risk.** If a major vendor (Anthropic, Microsoft, etc.) ships an official CLI-to-MCP tool before Portage matures, the project may need to pivot toward a specific niche (e.g., safety-focused wrapping, or man-page parsing specifically) rather than being the general solution.
- **Cloudflare Workers' native-binary limitation** means the "fully remote, one-click hosted" version is more complex than the local version; the MVP should prioritize local-first usage and treat remote hosting as a later phase.

---

## 9. Success Criteria

- **MVP success:** Portage can take at least 5 real-world CLIs it has never seen configured for, and correctly generate usable MCP tools for them with no manual per-tool code, verified by successfully calling those tools from Claude Desktop or Claude Code.
- **Adoption success:** Published to GitHub with a clear README, picked up by at least one MCP server registry/directory.
- **Safety success:** No tool executes without passing through argument validation and allow-list checks; a security-conscious reviewer would consider the default configuration safe to install.

---

## 10. Glossary (for anyone unfamiliar with the terms)

- **MCP (Model Context Protocol):** An open standard, created by Anthropic, that lets AI assistants like Claude connect to external tools and data sources in a consistent way.
- **CLI (Command-Line Interface):** A program you interact with by typing text commands (as opposed to clicking buttons in a graphical app).
- **`--help` output:** The text a well-behaved CLI tool prints when you run it with a `--help` flag, describing its options and usage.
- **Man page:** A traditional Unix/Linux manual page for a command, viewable by running `man <command>`.
- **stdio transport:** A way for an MCP server to communicate with a client over standard input/output, typically used for local, same-machine setups.
- **Streamable HTTP transport:** A way for an MCP server to communicate with a client over the network, needed for remote/hosted setups.
- **Allow-listing:** Restricting execution to only an explicitly approved set of commands/flags, rather than allowing anything by default.

---

## 11. Next Step

The next concrete step is to start Phase 1: build the `--help` parser and get a minimal local `stdio` MCP server running against a small set of real CLIs. This report is intended to be handed to another AI assistant (e.g., ChatGPT) to produce a sequence of concrete build prompts, which will then be executed step-by-step inside Claude Code.
