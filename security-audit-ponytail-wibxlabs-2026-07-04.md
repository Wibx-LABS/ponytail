# Security Audit — ponytail (Wibx-LABS fork)

- **Date:** 2026-07-04
- **Target:** `github.com/Wibx-LABS/ponytail` (fork of `DietrichGebert/ponytail`, MIT)
- **Audited commit:** `40e50d9e03242aa5dd53ac771950f9127362b25f`
- **Threat model:** host-attacking code (exfiltration, arbitrary shell, persistence,
  backdoor). Not code quality.
- **Nature:** Node.js Claude Code plugin — SessionStart/UserPromptSubmit/SubagentStart
  hooks that inject static context, read `settings.json`, write a flag file; plus an MCP
  server, commands, and skills.

## Scanners

| Scan | Tool | Result |
|------|------|--------|
| Malware signature | ClamAV 1.5.3 (DB current) | **0 infected** |
| SAST | semgrep 1.168.0 (`--config auto`) | 0 runtime-code findings; hits are CI `github-actions-mutable-action-tag` + a benchmark-only `dynamic-urllib-use` |
| Dependencies | osv-scanner 2.4.0 | no lockfile / **no dependency vulns**; root package has zero runtime deps |
| Manual review | all hooks, MCP server, scripts, package.json | **SAFE** |

## Manual review — verdict: SAFE TO RUN

- **No network egress** in any shipped runtime file. The only hardcoded URL is the repo's
  own GitHub URL. No `fetch`/`http`/`net`/telemetry.
- **No process spawning at consumer runtime.** The lone `spawnSync`
  (`scripts/publish-openclaw-skills.js`) is a maintainer publish script, not shipped in
  `package.json.files`, not a hook.
- **Hooks** confine reads/writes to standard config/state dirs; `settings.json` parsed in
  try/catch (BOM-stripped), only the `statusLine` field inspected — never executed. Mode
  strings validated against a fixed allowlist.
- **Statusline setup snippet** is gated by an `isShellSafe()` allowlist; a hostile install
  path falls back to manual-setup text. Snippet is proposed, not executed.
- **No install scripts** (`preinstall`/`postinstall`/`prepare`) in any of the 4
  package.json. MCP server is stdio-only, read-only tool, no network/shell.
- **No** sensitive-file access, **no** obfuscation (no eval/base64 blobs/dynamic remote require).

## Verdict

**CLEAR as source-of-trust.** Pin `40e50d9…`. Re-audit on upstream pulls (hooks execute
`node` on every session event, so diff future syncs).
