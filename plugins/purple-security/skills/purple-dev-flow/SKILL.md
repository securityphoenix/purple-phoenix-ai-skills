---
name: purple-dev-flow
description: End-to-end developer flow — index changed files, then scan via SAST/SCA (with remediation), Exploit Hunt, or Prometheus, gating Hunt on threat-model readiness (auto-triggers threat modeling if missing, warns before the slow path).
trigger: ["purple flow", "dev flow", "index and scan", "full scan", "scan and hunt", "run the full pipeline", "/purple-dev-flow"]
---

# Purple Dev Flow Skill

## Purpose

One command that takes a repo from "I just changed some files" to "here are the vulnerabilities and their fixes (or confirmed exploits)" — composing the existing `purple-graph-navigator` (indexing) and `purple-scan` (scanning) skills, plus a threat-modeling readiness gate for Hunt.

| Step | What happens |
|---|---|
| 1. Index | Ensure the workspace graph reflects the current changes (`analyze` if stale) |
| 2. Choose mode | SAST/SCA (fast, with remediation) · Exploit Hunt (deep, gated on threat model) · Prometheus (external surface) |
| 3. Gate (Hunt only) | Check `threatmodel_get_latest`; if missing/failed, warn it adds a slow step, then `threatmodel_run_assessment` |
| 4. Execute | Run the chosen scan/hunt/probe |
| 5. Remediate (SAST/SCA only) | Chain confirmed findings into `remediate_finding` / `remediate_batch` |

This skill does not replace `purple-scan` or `purple-graph-navigator` — it **orchestrates** them plus the threat-model gate they don't cover.

---

## MCP Servers

Connect both:
- **`phoenix-graph`** — for `analyze` (indexing/freshness) and `entry_points`/`impact` if you want to scope Hunt to high-risk paths first.
- **`phoenix-security`** — for `sast_scan`, `remediate_*`, `exploit_hunt_*`, `prometheus_*`, and `threatmodel_*` (all threat-modeling tools default to this endpoint — they are not in `McpToolGroups.GRAPH_ONLY`).

Legacy stdio (`purplephx`) install: all tools are on one server, use as-is.

---

## Step 1 — Resolve Target Repo + Workspace

Same resolution order as `purple-scan` Step 1:
1. Explicit in user message
2. Earlier in this conversation
3. MCP: `resolve_workspace_repo({workspaceId})` (works on both endpoints — it's a `PLUMBING` tool)
4. REST: `GET /api/workspaces`
5. Ask the user

You need **both** `repoName` and `workspaceId` — SAST/Hunt/Prometheus tools take `repoName`/`repository`, threat-modeling tools take `workspace_id`.

> **Preflight:** if MCP tools are absent from `tools/list` or the REST fallback returns 401, run `scripts/purple mcp doctor` (add `--write --repo <path>` to repair the repo's `.mcp.json`) before falling back.

Also resolve `caller_sub` — the workspace owner's user id (`threatmodel_*` tools return `not_authorized` if `caller_sub` doesn't match the workspace's owner). It is the subject claim of the token you authenticated with. If you don't know it, read it off an already-successful tool response in this conversation, or ask the user — never guess one.

---

## Step 2 — Index the Changes

Before scanning, make sure the graph reflects the current code (this *is* "indexing the changes" — PR/diff-scoped tools like `sast_scan(mode="PR")` and Hunt still need a graph that isn't stale):

```
tool: analyze
args:
  repoName: "<repoName>"
```

Skip this call if the graph was already confirmed fresh earlier in this conversation (e.g. by `purple-graph-navigator`). If `analyze` reports `graphStale: false` already, don't re-run it — it's a full re-index and isn't free.

---

## Step 3 — Choose the Mode

Ask the user, or infer from their message:

| Signal in the message | Mode |
|---|---|
| "scan", "sast", "sca", "vulnerabilities in this PR", nothing specified | **SAST/SCA** (default — fastest, always safe to run) |
| "hunt", "exploit", "0-day", "prove exploitability" | **Exploit Hunt** |
| "prometheus", "probe", "external surface" | **Prometheus** |

If genuinely ambiguous, ask: *"Scan for SAST/SCA findings with remediation (fast), run a full Exploit Hunt (slower, proves exploitability, needs a threat model), or a Prometheus surface probe?"*

---

## Step 4A — SAST/SCA + Remediation

```
tool: sast_scan
args:
  repoName: "<repoName>"
  mode: "PR"        # diff-aware; use "FAST" for a full-repo pass instead
```

For each finding you want fixed:

```
tool: remediate_finding
args:
  findingId: "<id>"
```

Or for many at once: `remediate_batch` with the finding id list. A proposed fix is quality-gated
before it is ever offered, and **never auto-merges** — present the patch and let the user apply it.

> **Automated fix patches are not enabled on any deployment yet.** For read-only guidance on a
> single SAST finding — what it is, where, the code, and how to fix it — use
> `finding_remediation` instead; that path works today. See the `/purple-scan` skill.

---

## Step 4B — Exploit Hunt (gated on Threat-Model readiness)

### 1. Check readiness

```
tool: threatmodel_get_latest
args:
  workspace_id: "<workspaceId>"
  tenant_id: "<tenantId>"
  caller_sub: "<callerSub>"
```

Interpret the result:

| Result | Meaning | Action |
|---|---|---|
| `status: "COMPLETED"` or `status: "DEGRADED"` | A threat model exists (DEGRADED is expected/normal for low-signal repos, not a failure) | Proceed straight to Hunt (below) |
| `error: "not_found"` | No run has ever been generated | Trigger one (step 2 below) |
| `status: "FAILED"` | Last run errored | Offer to retrigger (step 2), or proceed without one if the user prefers |
| `error: "threat_modeling_unavailable"` | Feature flag `features.threat-modeling` is off on this server | Tell the user threat-modeling isn't enabled here; ask whether to proceed with Hunt without it, or stop |

### 2. Trigger if missing (warn first — this is the slow path)

**Before calling, tell the user:** *"No threat model exists yet for this workspace. I'll generate one now using the deterministic engine (fast, free, graph-only) so Hunt has trust-boundary context. If you want deeper LLM-based STRIDE reasoning instead, say so — that engine is slower and uses metered LLM spend. Either way this adds a step before the hunt starts."*

```
tool: threatmodel_run_assessment
args:
  workspace_id: "<workspaceId>"
  tenant_id: "<tenantId>"
  caller_sub: "<callerSub>"
  scope: "WORKSPACE"
  # assessment_engine: "LLM"   # only if the user explicitly asked for the deeper/slower engine
```

This call is **synchronous** — it returns only once the run completes, there is no poll loop. If it returns `degraded_reasons`, mention them but still proceed (DEGRADED is a usable result, not a blocker).

> **Known limitation:** on current deployments `assessment_engine` may be ignored server-side (the run is always `DETERMINISTIC`). If you ask for `LLM` and the result looks graph-only, that is why — not a fault in this skill.

### 3. Run the hunt

```
tool: exploit_hunt_run
args:
  repository: "<repoName>"
  max_targets: 20
  budget_usd: 5.00
```

Then poll `exploit_hunt_status` exactly as in `purple-scan` Step 2A. The hunt's own tool loop pulls threat-model context automatically via its internal `get_threat_model` tool when `phx.hunt.trust-delta-enabled` is on — you don't need to pass the threat model into `exploit_hunt_run` yourself.

---

## Step 4C — Prometheus

Identical to `purple-scan` Step 2B — `prometheus_probe` → poll `prometheus_get_run`.

---

## Step 5 — Present Results

Use the same result-formatting conventions as `purple-scan` Step 4 (SAST table / Hunt confirmed-exploit blocks / Prometheus hypotheses), and additionally report the threat-model gate outcome for Hunt runs:

```
## Exploit Hunt — {repoName}

Threat model: {existing | generated just now (DETERMINISTIC|LLM)} — status {status}
Confirmed: {n} exploits | Cost: ${costUsd}
...
```

---

## MCP Tool Quick Reference (dev-flow specific — see `purple-scan` for the rest)

| Tool | Endpoint | Purpose | Key args |
|---|---|---|---|
| `analyze` | `phoenix-graph` | (Re)index a repo | `repoName` |
| `threatmodel_get_latest` | `phoenix-security` | Check threat-model readiness | `workspace_id`, `tenant_id`, `caller_sub` |
| `threatmodel_run_assessment` | `phoenix-security` | Trigger a threat-model run (synchronous) | `workspace_id`, `tenant_id`, `caller_sub`, `scope`, `assessment_engine` |
| `sast_scan` | `phoenix-security` | SAST/SCA scan | `repoName`, `mode` |
| `remediate_finding` / `remediate_batch` | `phoenix-security` | Propose fixes | `findingId` |
| `exploit_hunt_run` / `exploit_hunt_status` | `phoenix-security` | Run/poll Hunt | `repository` / `hunt_run_id` |
| `prometheus_probe` / `prometheus_get_run` | `phoenix-security` | Run/poll Prometheus | `tenant_id`, `repo_name` / `run_id` |

---

## Error Handling

| Error | Cause | Resolution |
|---|---|---|
| `threatmodel_*` → `not_authorized` | `caller_sub` doesn't match workspace owner | Re-resolve `caller_sub` from your token's subject claim, or ask the workspace owner |
| `threatmodel_*` → `threat_modeling_unavailable` | `features.threat-modeling=false` server-side | Proceed without a threat model (note the limitation) or ask an admin to enable the flag |
| Hunt runs but ignores the threat model | `phx.hunt.trust-delta-enabled=false` | Expected — the flag defaults off; the gate you ran still improves standalone `get_threat_model` context, just not privilege-delta gating |
| `graphStale: true` after `analyze` | Re-index still in progress or failed | Re-run `analyze`, check `/api/analyze/status` |
| Any `sast_scan`/`exploit_hunt_*`/`prometheus_*` error | See `purple-scan` skill's Error Handling table | — |
