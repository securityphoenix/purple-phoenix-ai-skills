# Changelog

All notable changes to the Phoenix Purple skills are documented here.

Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Versioning follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

The version here tracks the **skills and plugin**, not the Phoenix Purple backend.

---

## [1.1.0] — 2026-09-08

First public release of the Purple skills as a standalone, installable package.

### Added

- **Claude Code plugin and marketplace.** `purple-security@phoenix-purple` installs all three
  skills in two commands — no copying files into your project.
- **MCP configuration for four assistants** — Claude Code, Cursor, VS Code and Codex. Every one
  reads the token from the environment or prompts for it securely, so no configuration file in
  this repo ever holds a credential.
- **Codex support.** Codex has no plugin marketplace, so it gets an equivalent MCP registration
  for `~/.codex/config.toml`.
- **Per-finding remediation guidance in `purple-scan`.** Ask about one SAST finding and get what
  it is, where it is, the surrounding code, and how to fix it — with an explicit
  `Remediation Availability` line so "no fix text" is never confused with "no fix needed".
- **Code snippets in findings lists.** `sast_findings` can now attach the code for each finding.
  Total snippet size is capped, and the response says so when the cap is reached rather than
  silently dropping code.

### Fixed

- **All three skills were loading without their name and description.** A formatting error in each
  skill's header meant every header field was discarded when the skill loaded. Corrected.
- **`purple-scan` named the wrong identifier for remediation lookups.** A SAST finding carries two
  ids, and the documentation pointed at the one that cannot be used to look up its remediation —
  so the lookup could never succeed. It now names the right one and explains the difference.

### Changed

- **`purple-scan` separates remediation guidance from automated fix patches.** These are different
  features with different readiness: guidance works today; generated fix patches are not enabled
  on any deployment yet. The skill previously led with the one that does not work.

---

## Earlier

These skills shipped inside the Phoenix Purple installation pack before this repository existed.
Their history is not reproduced here; this changelog starts at the first standalone release.
