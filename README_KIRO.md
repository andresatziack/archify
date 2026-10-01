# Archify in Kiro — Install, Use, and Security Notes

This guide explains how to **install the Archify skill manually inside Kiro** (without using the `npx skills add` installer), how to use it, and summarizes the **security review** of the project.

Archify is an **Agent Skill**: a `SKILL.md` instruction file plus a zero-dependency Node.js CLI (`bin/archify.mjs`) that turns a plain-language description or repository evidence into a self-contained, interactive **HTML diagram** (architecture / workflow / sequence / dataflow / lifecycle).

---

## 1. Prerequisites

- **Node.js >= 18** on your machine (the skill's CLI runs on Node). Check with `node --version`.
- A local copy of this repository (you already have it).
- No `npm install` is required to *use* the skill. The runtime ships with **zero production dependencies**; the four packages in `package.json` are `devDependencies` used only for building/testing the project, not for generating diagrams.

---

## 2. How Kiro loads skills (the mechanism)

Kiro discovers skills from two locations:

| Scope | Location | When to use |
|---|---|---|
| **User (global)** | `~/.kiro/skills/<skill-name>/` | Available in every workspace on your machine |
| **Workspace (project)** | `<project>/.kiro/skills/<skill-name>/` | Available only inside this specific project |

A skill is simply **a folder containing a `SKILL.md` file** (with YAML frontmatter). Kiro reads the frontmatter `name` and `description` to decide when the skill is relevant, then follows the instructions in the body. Archify's `SKILL.md` already has the required frontmatter:

```yaml
---
name: archify
description: Create polished, validated architecture, workflow, sequence, data-flow, and lifecycle/state diagrams as explorable standalone HTML...
license: MIT
metadata:
  version: "3.0"
  author: tt-a1i
---
```

So installing manually just means **placing the skill folder where Kiro looks for it**, no npm registry involved.

---

## 3. Manual install into Kiro (instead of `npx skills add`)

The skill package is the inner **`archify/`** directory of this repository (the one that contains `SKILL.md`, `bin/`, `renderers/`, `schemas/`, `references/`, and `examples/`).

> Paths below assume the repository is checked out and `<repo>` is its root. Pick **either** the global **or** the workspace option.

### Option A — Global install (available in all workspaces)

**macOS / Linux:**
```bash
mkdir -p ~/.kiro/skills
cp -R <repo>/archify ~/.kiro/skills/archify
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force "$HOME\.kiro\skills" | Out-Null
Copy-Item -Recurse "<repo>\archify" "$HOME\.kiro\skills\archify"
```

### Option B — Workspace install (only this project)

**macOS / Linux:**
```bash
mkdir -p <project>/.kiro/skills
cp -R <repo>/archify <project>/.kiro/skills/archify
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force "<project>\.kiro\skills" | Out-Null
Copy-Item -Recurse "<repo>\archify" "<project>\.kiro\skills\archify"
```

### Option C — Symlink (keep it updated from your checkout)

If you want the installed skill to track your local git checkout, symlink it instead of copying. Use a **durable** checkout path (never a temp/throwaway folder):

```bash
ln -s <repo>/archify ~/.kiro/skills/archify        # global
# or
ln -s <repo>/archify <project>/.kiro/skills/archify # workspace
```

### Verify the install

The target folder must contain `SKILL.md` at its top level:

```bash
ls ~/.kiro/skills/archify/SKILL.md    # global
# or
ls <project>/.kiro/skills/archify/SKILL.md  # workspace
```

Then open Kiro in the relevant workspace. The skill is active when Kiro can see that `SKILL.md`. You can confirm the CLI runs:

```bash
node ~/.kiro/skills/archify/bin/archify.mjs doctor
node ~/.kiro/skills/archify/bin/archify.mjs demo ./archify-demo-out
```

---

## 4. Using Archify in Kiro

Once installed, just ask Kiro in natural language. Examples:

```text
Use Archify to diagram a web request: Browser calls the API, the API checks
Redis, and a cache miss queries PostgreSQL and fills the cache.
```

```text
Analyze this repository, then use Archify to create a high-level runtime
architecture diagram with 8–12 core components and one primary path.
```

Then iterate: “Add authentication”, “Highlight the cache-miss path”, “Switch to the light theme”.

Each request produces a self-contained `.html` file (plus a validation receipt) in a per-request folder like `.archify/<type>-<slug>-<timestamp>/`. The HTML opens in any browser with no Archify installation required to view it.

---

## 5. Updating and uninstalling

- **Update:** replace the installed folder with a newer `archify/` directory (re-run the copy in step 3), or `git pull` in your checkout if you used the symlink option.
- **Uninstall:** delete the folder.
  ```bash
  rm -rf ~/.kiro/skills/archify          # global
  rm -rf <project>/.kiro/skills/archify  # workspace
  ```
- **Disable the optional update check / all networking:** set the environment variable `ARCHIFY_UPDATE_CHECK_DISABLED=1`. See the security notes below for what the update check does.

---

## 6. Security review summary

A manual source review was performed on the skill package (`archify/`), its CLI (`bin/`), renderers, scripts, and the browser-side viewer. **No malicious code was found.** The project is unusually security-conscious. Findings:

### No install-time code execution
- **No `preinstall` / `postinstall` / `prepare` lifecycle hooks** in any `package.json`. Copying the folder into Kiro runs nothing. (This is the most common vector for malicious npm packages, and it is absent.)

### No obfuscation or credential access
- No `eval`, no `new Function`, no obfuscated/encoded payloads. The only `Buffer.from(..., 'base64')` decodes a screenshot during export (legitimate).
- No access to SSH keys, `/etc/passwd`, environment secrets/tokens, cloud credentials, or crypto wallets.

### Dependencies
- The skill runs with **zero runtime dependencies**. `package.json` lists only four **devDependencies** (`ajv`, `parse5`, `saxes`, `simple-icons`), all reputable and **pinned with integrity hashes** in `package-lock.json`. These are not needed to generate diagrams.

### Network behavior — transparent and limited
- The **only** outbound network activity in normal use is an **optional update check**: a single `GET` of a fixed, hardcoded manifest URL (`https://tt-a1i.github.io/archify/skill-updates/archify/stable.json`).
  - It runs in a **separate, sandboxed child process** with a ~1-second deadline, is **SIGKILL-ed on timeout**, caps the response at 32 KB, and only reads back a strictly validated JSON manifest.
  - It **never downloads or executes an update** — it only decides whether to *show you a reminder*. You choose if/when to update.
  - It sends **no** version, prompt, project data, account/device ID, or ETag — only normal HTTP metadata (your IP and the time).
  - Fully disableable with `ARCHIFY_UPDATE_CHECK_DISABLED=1`.
- **Optional brand-mark fetching** (only when you explicitly ask for a brand mark by URL) includes strong **anti-SSRF / anti-DNS-rebinding** protections: it blocks private/loopback/link-local IPv4 and IPv6 ranges (including IPv4-mapped addresses), rejects embedded credentials and non-standard ports, pins the validated IP into the socket, limits redirects, bounds body sizes, and rejects compressed encodings.
- The **preview server** binds to **loopback only** (`127.0.0.1`, OS-assigned port) and the file opener restricts URLs to `http://127.0.0.1:<port>/`. It is not exposed to the network.

### Filesystem behavior — scoped and safe
- Diagram output is written only to the working/output directory you choose, using no-clobber (`wx`) flags, `0o700` directory permissions, and atomic rename/`linkSync` publish patterns. No writes to home config, SSH, or system paths.

### Subprocess usage — no command injection
- All child processes spawn either Node itself (`process.execPath`) or `git` with fixed, non-interpolated arguments. The platform file opener uses `shell: false` and passes the target path via a child-only environment variable instead of string-interpolating it into a command.

### Browser-side viewer — clean
- The generated HTML and viewer scripts contain **no `innerHTML`/`document.write`/`eval`/`new Function`** and **no `fetch`/XHR/WebSocket** exfiltration. The only browser storage use is a single `localStorage` key remembering your reduced-motion preference, wrapped in try/catch.

### Conclusion
> **Safe to install and run.** No malicious code, no hidden network exfiltration, no install hooks, and defense-in-depth around the few features that do touch the network or filesystem. If you want zero outbound traffic, set `ARCHIFY_UPDATE_CHECK_DISABLED=1`.

*This review reflects the source at the current checkout. Re-review after pulling significant updates, and prefer installing from a trusted, pinned source.*

---

## 7. Quick reference

```bash
# Install (global)
cp -R <repo>/archify ~/.kiro/skills/archify

# Verify
ls ~/.kiro/skills/archify/SKILL.md
node ~/.kiro/skills/archify/bin/archify.mjs doctor

# Disable all networking / update reminders
export ARCHIFY_UPDATE_CHECK_DISABLED=1

# Uninstall
rm -rf ~/.kiro/skills/archify
```
