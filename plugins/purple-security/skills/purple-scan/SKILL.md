---
name: purple-scan
description: Unified Phoenix Purple security scanning skill — PR scan + AI validation (scan-assess), 0-day exploit hunt, or Prometheus surface probe — results and remediation surfaced here via MCP tools or REST fallback.
trigger: ["purple scan", "scan-assess", "scan pr", "hunt vulnerabilities", "prometheus probe", "0-day hunt", "purple assess", "/purple-scan"]
---

# Purple Scan Skill

## Purpose

Run Phoenix Purple security scanning and surface findings + remediation in this conversation.

| Mode | Trigger | Primary path |
|------|---------|-------------|
| **hunt** | "hunt", "0-day", "exploit hunt" | MCP: `exploit_hunt_run` → `exploit_hunt_status` |
| **prometheus** | "prometheus", "surface probe", "promytheus" | MCP: `prometheus_probe` → `prometheus_get_run` |
| **scan-assess** | "scan PR", "assess PR", "scan-assess" | REST: resolve → execute → poll (no MCP tool yet) |
| **scan-all** | "scan everything", "full scan", "scan all domains", "purple scan all" | MCP: `scan_all` (single call, ordered multi-domain bundle) |
| **scan-domain** | "run SAST", "scan for secrets", "check IaC", "scan the containers", "0-day scan" (any request naming ONE domain) | MCP: the individual scanner tool for that domain (Step 2E) — **not** `scan_all` |

---

## MCP Server

**Connect `phoenix-security`** — the security scanning endpoint (`/api/v1/external/mcp/security`).
It exposes all security tools (`exploit_hunt_*`, `prometheus_*`, `sast_*`, `scaffold_*`, etc.)
plus the 2 plumbing tools. Graph-navigation tools (`analyze`, `query`, `context`, `impact`,
`entry_points`, etc.) are **not** available on this endpoint.

| Install type | Server to use |
|---|---|
| HTTP split install (recommended) | `phoenix-security` |
| Legacy stdio (`purplephx`) | All tools available — use as before |

> **Need graph context first?** Connect `phoenix-graph` (or invoke the `purple-graph-navigator`
> skill) to explore the call graph, entry points, or blast radius before starting a hunt.
> Graph tools and security tools run from separate endpoints in the split install.

---

## Step 0 — Detect MCP vs REST

**Check MCP first.** The `phoenix-security` MCP server exposes all security tools directly — no JWT, no curl, no environment setup needed when it is connected.

```
phoenix-security MCP connected?  →  use MCP tool calls (Steps 2A/2B/2D below)
MCP not configured?               →  fall back to REST+curl (Step 3 below)
```

To check: call `tools/list` or try `resolve_workspace_repo`. If it returns data, MCP is live.

> **Preflight:** if MCP tools are absent from `tools/list` or the REST fallback returns 401, run `scripts/purple mcp doctor` (add `--write --repo <path>` to repair the repo's `.mcp.json`) before falling back.

> **REST path — authentication:** the REST fallback auto-authenticates via the credential stored by `scripts/purple auth login`. Run `scripts/purple auth token` to obtain the current bearer token (resolved and auto-refreshed from `~/.phoenix/credentials.json`). If that command fails, run `scripts/purple auth login --tenant-url <your-tenant>` first.

---

## Step 1 — Resolve Target Repo

Check in this order — stop as soon as one resolves:

1. Explicit in user message (e.g. "hunt in `VulnerableApp`")
2. Earlier in this conversation (repo name already established)
3. Via MCP: `resolve_workspace_repo({workspaceId})` or read catalog via `phoenix-graph` if connected
4. Via REST: `GET /api/workspaces` → show repo list
5. Ask the user

For `scan-assess` also resolve `baseBranch` (default `main`) and `headBranch`.

---

## Step 2A — Exploit Hunt via MCP

### Seed + start in one call

```
tool: exploit_hunt_run
args:
  repository: "<repoName>"
  max_targets: 20          # increase for deep scan
  budget_usd: 5.00
```

Returns: `huntRunId`, `status`, `totalTargets seeded`

### Poll status

```
tool: exploit_hunt_status
args:
  hunt_run_id: "<huntRunId>"
```

Poll every 10–15 s until `status` is `COMPLETE` or `FAILED`.
Report progress: `processedTargets / totalTargets`, `confirmedFindings`, `cost so far`.

### Need graph context before hunting?

Graph tools (`entry_points`, `key_functions`, `impact`) live on the `phoenix-graph` endpoint,
not here. If the user wants to scope the hunt to high-risk entry points first:

1. Switch to `purple-graph-navigator` (or connect `phoenix-graph`) and run `entry_points` + `key_functions`.
2. Return here to seed `exploit_hunt_run` with the interesting file paths from that output.

---

## Step 2B — Prometheus Probe via MCP

Prometheus requires a `tenant_id`. It is your organization's id on the Phoenix tenant — the subject claim of the token you authenticated with. If you do not know it, ask your Phoenix administrator rather than guessing; a wrong `tenant_id` is rejected, never silently substituted.

```
tool: prometheus_probe
args:
  tenant_id: "<tenantId>"
  repo_name: "<repoName>"
  scope: "full"
  budget_usd: 2.00
  max_loops: 8
  surface_filter: "<optional — e.g. 'injection'>"
```

Returns: `run_id`, `status`, `hypothesis_tree_summary`, `top_hypotheses[3]`, `top_results[3]`, `cost_used_usd`, `source_of_defaults`

### Poll for results

```
tool: prometheus_get_run
args:
  tenant_id: "<tenantId>"
  run_id: "<runId>"
```

Poll every 15 s until `status` is `COMPLETED` or `FAILED` or `CANCELLED`.
Report each poll: iteration, active hypotheses, cost.

### To cancel

```
tool: prometheus_cancel_run
args:
  tenant_id: "<tenantId>"
  run_id: "<runId>"
```

---

## Step 2C — Graph Context for Security Questions

Graph tools (`entry_points`, `key_functions`, `impact`, `context`, `overview`, `processes`) are
on the `phoenix-graph` endpoint and are **not callable from `phoenix-security`**.

For graph-backed security questions alongside a scan:

- **Both MCP servers connected:** call graph tools via `phoenix-graph`, security tools via `phoenix-security` — Claude Code routes each call to the right server automatically.
- **Only `phoenix-security` connected:** use REST fallback for graph context (see Step 3 auth), or ask the user to also connect `phoenix-graph`.
- **Focused graph exploration first:** switch to `purple-graph-navigator` skill, complete navigation, then return here.

Recommended: "entry points + blast radius → seed hunt" workflow uses both servers.

---

## Step 2D — Multi-Domain Scan (`scan_all`) via MCP

`scan_all` runs an ordered bundle of scan phases against one resolved repository in a single call:
graph freshness, SAST, SCA, secret, container, IaC, and assessment run by default; exploit hunt,
Prometheus, zero-day, and live DAST are opt-in only (see the Default set below).

**Use `scan_all` for "give me the standard sweep."** If the user names one specific domain, wants
a non-default mode/depth on just that domain, or wants the fastest/cheapest possible answer for
one scanner, call that scanner directly instead — see **Step 2E** below. Wrapping a single-domain
request in `scan_all` (`include: ["sast"]`) works, but pays the bundle's dispatch/budget machinery
for one tool when the direct call is simpler and identical in effect.

### Per-domain params: flat args or the `domains{}` envelope

Every domain's parameters can be supplied either as legacy flat top-level args (`sast_mode`,
`paths`, `scanMode`, `workspaceId`) or nested under a `domains` object keyed by domain
(`domains.sast.mode`, `domains.sca.mode`, `domains.secret.baseRef`, `domains.container.mode`,
`domains.iac.scanMode`, `domains.iac.paths`, `domains.zeroday.workspaceId`, `domains.dast.target_id`, …).

**`scan_all` names the repo `repoName`, not `repository`.** Unlike `scan_sca` / `scan_secret` /
`container_scan`, which accept `repository` as a real alias, `scan_all` requires `repoName` and
throws if given `repository`.

`domains.sca.concurrency` is accepted by the schema but has **no effect through `scan_all`** — it
only applies to a workspace-scoped SCA scan, a different entry point. `domains.sca.mode` through
`scan_all` selects the interactive scan depth and defaults to `FAST`.

**Precedence rule:** when both a `domains.<domain>` field and its flat-arg equivalent are supplied
for the same call, **`domains.<domain>` always wins, deterministically.**

### `include[]` — which phases run

```
tool: scan_all
args:
  repoName: "<repoName>"        # NOT `repository` — scan_all has no `repository` alias
  tenant_id: "<tenantId>"
  include: ["sast", "sca", "secret", "container", "iac"]   # optional, see defaults below
  dry_run: false
```

- **Default set (used when `include` is omitted):** `sast`, `sca`, `secret`, `container`, `iac`,
  `assessment`.
- **Opt-in only — never run unless explicitly named in `include`:** `hunt`, `prometheus`,
  `zeroday`, `dast`. These are expensive and/or high-signal domains.
  - `zeroday` additionally requires a `workspaceId` (top-level `workspaceId` or
    `domains.zeroday.workspaceId` — the latter wins) because zero-day findings are
    workspace-scoped. Absent → the phase reports `skipped`; a workspace is never synthesized.
  - `dast` additionally requires `domains.dast.target_id` (no flat-arg equivalent, and
    deliberately no `url` argument anywhere — see below). Absent → `skipped`.
- `dry_run: true` returns only the planned step list (`steps[]`: `tool`, `description`,
  `available`) without executing anything.

### Phase statuses — read every result's `status`

Each phase in the response's `results[]` carries **exactly one** of four statuses:

| Status | Meaning |
|--------|---------|
| `completed` | The phase ran to completion and its result is attached. |
| `incomplete` | The phase did **not** finish inside the interactive wait budget (server-configured, ~45s by default) before this call returned. **This is not "nothing found" — do not report it as zero findings.** The phase keeps running in the background; poll its own tool (e.g. `sast_findings`, `iac_findings`) for the eventual result. |
| `failed` | The phase was attempted and threw — the result carries a `reason`. |
| `skipped` | The phase was never attempted at all — either its collaborator isn't wired in this runtime, or a required precondition was structurally absent (e.g. `zeroday`'s `workspaceId`, `dast`'s `target_id`). **`skipped` always carries a `reason`.** |

Always surface `incomplete` and `skipped` phases to the user by name and reason — never collapse
either into "0 findings" for that domain.

### `dast_scan` / the `dast` phase — deliberately narrow

Both the standalone `dast_scan` tool and `scan_all`'s `dast` phase launch a **live scan against a
real external host** and are treated accordingly:

- **Flag-gated, opt-in only.** Requires `phx.features.external-assessment` AND
  `phx.dast.mcp-scan-enabled` (both default off in every profile) plus the caller org's
  `webApiDast` entitlement.
- **No `url` argument, by design.** The only addressing argument is `target_id` — a target the
  organization already registered and had its host ownership `VERIFIED`. A model can never name
  its own egress target; supplying `url`/`target_url`/`targetUrl`/`host` refuses the whole call.
- **Access gate is org-membership only.** Any caller holding a valid API token for the org can
  invoke it once the flags/entitlement/target checks pass — there is deliberately no additional
  per-user role or permission check on this transport (it has no per-user role data to check
  against).
- The scan is asynchronous — the call returns an `assessment_id` to poll, never a verdict.

---

## Step 2E — Individual Scanner Calls via MCP

**When to use this instead of `scan_all` (Step 2D):** the user names one specific domain
("just run SAST", "check for secrets", "scan the Dockerfiles", "run a 0-day scan"); a non-default
mode/depth is needed on that one domain while everything else stays untouched (e.g. `DEEP` SAST
without triggering SCA/secret/container/IaC too); or the fastest/cheapest possible single-domain
answer matters more than the bundle. `scan_all` is for "give me the standard sweep in one call";
these tools are for "run exactly this one thing, exactly how I want it."

Each domain follows the same shape: **trigger** the scan, then **poll/read** its own findings
tool. All accept `tenant_id`. Most accept `repository`/`repoName` — check the per-tool row in the
"MCP Tool Quick Reference" table further down this file; `scan_all` is the one exception with no
`repository` alias (see Step 2D).

### SAST
```
tool: sast_scan
args:
  repoName: "<repoName>"
  mode: "FAST"           # FAST | SMART | DEEP | PR
  paths: ["src/"]         # optional — scope to a subtree
  tenant_id: "<tenantId>"
```
Poll/read: `sast_findings({ repoName, tenant_id, workspaceId, severity, status, ruleId, limit,
offset, includeSnippet })`. Each finding renders **two distinct ids** — an `ID:` line (the volatile
scan id) and, when the row has one, a separate `Stable ID:` line (the persisted `stable_finding_id`,
which survives an analyzer restart). They are never the same value. It also renders a `What:` line
(the finding's description) and, when the remediation ladder resolves guidance for it, a
`Fix: <text> (source: <rung>)` line — `Fix` is omitted entirely (never a placeholder) when nothing
resolves, so a missing `Fix` means "nothing resolved", never "no fix needed". Pass
`includeSnippet: true` (default `false`) to attach each finding's code snippet; total snippet bytes
across the response are capped, and the response states explicitly once the cap is hit and later
findings went without one.

For a single finding's what/where/snippet/fix instead of re-listing the whole page, use
`finding_remediation({ findingId, tenant_id, workspaceId })` — passing the **`Stable ID:`** value,
not the `ID:` value. That response always ends with `Remediation Availability: RESOLVED |
NONE_FOUND`, so an absent `Fix` is never ambiguous with "not checked". `finding_remediation` is
**SAST-only**; a findingId from any other class reports the same "not found" message an unknown id
does. Pass `workspaceId` when the repo is checked out into more than one workspace — an ambiguous
id is refused, not guessed.

`sast_finding_context` reads a **third, volatile in-memory** id space and does not accept either of
the ids above — do not chain `sast_findings` into it; use `finding_remediation`.

### SCA / SBOM
```
tool: scan_sca
args:
  repository: "<repoName>"    # or repoName
  mode: "QUICK"          # QUICK | FULL
  tenant_id: "<tenantId>"
```

### Secrets
```
tool: scan_secret
args:
  repository: "<repoName>"    # or repoName
  baseRef: "<baseBranch>"      # optional — scope to a diff instead of the whole repo
  scanners: ["gitleaks", "trufflehog"]
  tenant_id: "<tenantId>"
```

### Container
```
tool: container_scan
args:
  repository: "<repoName>"
  mode: "FAST"            # FAST | SMART | DEEP
  tenant_id: "<tenantId>"
```
Related: `container_taint_paths` (Dockerfile instruction data-flow), `container_base_images`
(CVE counts + freshness per base image).

### IaC
```
tool: iac_scan
args:
  repo_name: "<repoName>"
  scanMode: "FAST"         # FAST | SMART | DEEP | PR | IAC_ONLY | AI_ASSESSMENT
  paths: ["infra/"]         # optional
  tenant_id: "<tenantId>"
```
Poll/read: `iac_findings`. Related: `iac_assets`, `iac_controls`, `iac_taint_trace`,
`iac_toxic_combos`, `iac_module_check`.

### Zero-day
```
tool: zero_day_scan
args:
  repoPath: "<repoPath>"
  mode: "RECENT_COMMITS"       # PR_DIFF | RECENT_COMMITS | FULL_REPO
  workspaceId: "<workspaceId>"  # required — zero-day findings are workspace-scoped, never synthesized
  tenant_id: "<tenantId>"
```
Poll/read: `zero_day_findings({ tenant_id, workspaceId })`.

### DAST
Same tool whether called standalone or as `scan_all`'s `dast` phase — see Step 2D's
`dast_scan` subsection above for the full gate chain (flags, `VERIFIED`-target-only, no `url` arg).

### Assessment (OWASP/ASVS)
```
tool: run_assessment
args:
  repoName: "<repoName>"
  tenant_id: "<tenantId>"
  scope: "FULL"          # FULL | PATH
  targetPath: "src/auth/"  # optional — required only when scope=PATH
  depth: "STANDARD"        # LIGHT | STANDARD | DEEP
```
`run_assessment` is the compat alias that resolves `repoName` → the indexed repo path for you.
The underlying `security_assessment` tool exists too, but takes a raw filesystem `repoPath`
(not a repo name) — use `run_assessment` unless you already have that path resolved.

**Statuses:** an individual scanner call returns its own result shape (not `scan_all`'s
`results[]` envelope), but the same discipline applies — a scan still in progress is never
"0 findings"; poll the domain's findings tool rather than reporting an empty result as final.

---

## Step 3 — REST Fallback (when MCP not configured)

> **This path needs the Phoenix Purple CLI (`scripts/purple`), which ships with the full Phoenix
> installation pack — not with the skills package.** If you installed the skills on their own, the
> MCP path above is the supported route; configure MCP rather than falling back here.

### Auth resolution (check before asking)

| What you need | Check order |
|--------------|-------------|
| Base URL | `$PURPLE_BASE_URL` → `$PHX_BASE_URL` → ask. Never guess a tenant host |
| JWT | resolved automatically via `scripts/purple auth token` (see below) |
| LLM key (AI modes only) | `$PHX_LLM_API_KEY` (server-side, skip header) → `$OPENAI_API_KEY` → `$ANTHROPIC_API_KEY` → ask |

Some deployments set the LLM key server-side; where they do, no BYOK header is needed.

```bash
# Token is resolved + auto-refreshed from ~/.phoenix/credentials.json by the wrapper.
# If this errors, run:  scripts/purple auth login --tenant-url <your-tenant>
JWT="$(scripts/purple auth token)" || { echo "Not authenticated — run: scripts/purple auth login"; exit 1; }
BASE="${PURPLE_BASE_URL:-${PHX_BASE_URL:?set PURPLE_BASE_URL or PHX_BASE_URL to your tenant URL}}"
```

### PR Scan (scan-assess) — REST only

```bash
# 1. Resolve bundles
RESOLVE=$(curl -s -X POST "${BASE}/api/v1/pr-scan/resolve" \
  -H "Authorization: Bearer ${JWT}" \
  -H "Content-Type: application/json" \
  -d '{"repoName":"<repo>","workspaceId":"<wsId>"}')

# 2. Execute (add x-llm-api-key only for AI tiers)
JOB=$(curl -s -X POST "${BASE}/api/v1/pr-scan/execute" \
  -H "Authorization: Bearer ${JWT}" \
  ${LLM_API_KEY:+-H "x-llm-api-key: ${LLM_API_KEY}"} \
  -H "Content-Type: application/json" \
  -d '{
    "repoName":"<repo>",
    "baseBranch":"<base>",
    "headBranch":"<head>",
    "resolvedBundles":'"$(echo $RESOLVE | jq '.resolvedBundles')"',
    "timeoutSeconds":300
  }')

JOB_ID=$(echo $JOB | jq -r '.jobId')

# 3. Poll
while true; do
  S=$(curl -s "${BASE}/api/v1/pr-scan/${JOB_ID}" -H "Authorization: Bearer ${JWT}" | jq -r '.status')
  [[ "$S" == "COMPLETE" || "$S" == "FAILED" || "$S" == "BLOCKED" ]] && break
  sleep 5
done

# 4. Fetch SARIF
curl -s "${BASE}/api/v1/pr-scan/${JOB_ID}/sarif" -H "Authorization: Bearer ${JWT}"
```

### Hunt via REST (alternative to MCP)

```bash
HUNT=$(curl -s -X POST "${BASE}/api/ctf-hunt/start" \
  -H "Authorization: Bearer ${JWT}" \
  ${LLM_API_KEY:+-H "x-llm-api-key: ${LLM_API_KEY}"} \
  -H "Content-Type: application/json" \
  -d '{"repoName":"<repo>","budget":5.00,"maxFiles":20,"mode":"NORMAL"}')

RUN_ID=$(echo $HUNT | jq -r '.runId')
STREAM_SECRET=$(echo $HUNT | jq -r '.streamSecret')

# Stream (background)
curl -sN "${BASE}/api/ctf-hunt/stream/${RUN_ID}?stream_secret=${STREAM_SECRET}" \
  -H "Authorization: Bearer ${JWT}" &

# Collect results + fix patches
RESULTS=$(curl -s "${BASE}/api/ctf-hunt/results/${RUN_ID}" -H "Authorization: Bearer ${JWT}")
for ID in $(echo $RESULTS | jq -r '.exploits[] | select(.verdict=="CONFIRMED") | .id'); do
  curl -s "${BASE}/api/risks/exploits/${ID}/detail" -H "Authorization: Bearer ${JWT}" \
    | jq '{title, cwe, severity, fixPatch, proofOfConcept}'
done
```

---

## Step 4 — Present Results

### Hunt results

```
## Exploit Hunt — {repoName}

Confirmed: {n} exploits  |  Cost: ${costUsd}  |  Files: {n} scanned
Pass breakdown: HUNT {n} → JUDGE confirmed {n} → VERIFY exploitable {n}

### Confirmed #{n} — {title} ({severity}, {cwe})
- File: {filePath}:{line}
- Judge: CONFIRMED — confidence {pct}%, feasibility {HIGH|MEDIUM|LOW}
- PoC: `{proofOfConcept}`
- Remediation: {suggestedFix}

Fix patch:
```diff
{fixPatch}
```
```

### Prometheus results

```
## Prometheus Probe — {repoName}

Status: {status}  |  Loops: {n}/{max}  |  Cost: ${costUsd}
Hypotheses: {total} generated, {confirmed} confirmed

### Hypothesis #{n} — {claim} ({cwe}, confidence {pct}%)
- Surface: {surface}
- Evidence: {evidenceSummary}
- Attack path: {attackPath}
- Remediation hint: {remediationHint}
```

### scan-assess results

```
## PR Scan — {headBranch} → {baseBranch}

Verdict: PASS | WARN | BLOCK
Tiers: {list}  |  Graph: fresh | ⚠ stale

| # | Severity | CWE | Rule | File:Line | AI-validated |
|---|----------|-----|------|-----------|--------------|

### Critical — {cwe}: {ruleId}
- File: {filePath}:{line}
- Why exploitable: {aiRationale}
- Remediation: {suggestedFix}
```

---

## Step 5 — Follow-up offers

```
Next steps:
1. Re-run in DEEP mode for full PoC verification (Hunt)
2. Hand Prometheus hypotheses to Exploit Hunt for PoC confirmation
3. Apply fix patches (Hunt confirmed exploits)
4. Export SARIF to GitHub Security tab (scan-assess)
5. Graph context: connect phoenix-graph and ask "who calls {function}?" or "blast radius of {symbol}?"
6. Set up scheduled weekly hunt
```

---

## MCP Tool Quick Reference (`phoenix-security` endpoint)

| Tool | Purpose | Key args |
|------|---------|----------|
| `exploit_hunt_seed` | Seed targets only | `repository`, `max_targets`, `budget_usd` |
| `exploit_hunt_run` | Seed + start hunt | `repository`, `max_targets`, `budget_usd` |
| `exploit_hunt_status` | Poll hunt progress | `hunt_run_id` |
| `exploit_hunt_ingest` | Ingest findings into a run | `hunt_run_id`, `findings[]` |
| `prometheus_probe` | Start Prometheus probe | `tenant_id`, `repo_name`, `budget_usd`, `max_loops` |
| `prometheus_get_run` | Get run + hypotheses | `tenant_id`, `run_id` |
| `prometheus_cancel_run` | Cancel a probe | `tenant_id`, `run_id` |
| `sast_scan` | Run SAST scan — see Step 2E | `repoName`, `mode` (FAST/SMART/DEEP/PR), `paths[]`, `tenant_id` |
| `sast_findings` | Fetch SAST findings (persisted store; renders both an `ID` and a separate `Stable ID`) — see Step 2E | `repoName`, `tenant_id`, `workspaceId`, `severity`, `status`, `ruleId`, `limit`, `offset`, `includeSnippet` |
| `finding_remediation` | Single SAST finding's what/where/snippet/fix by its `Stable ID`; always states `Remediation Availability`. SAST only — see Step 2E | `findingId` (= `Stable ID`), `tenant_id`, `workspaceId` |
| `scan_all` | Multi-domain scan bundle in one call (default: SAST/SCA/secret/container/IaC/assessment; opt-in: hunt/prometheus/zeroday/dast) — see Step 2D | `repoName` (**not** `repository` — `scan_all` has no `repository` alias, unlike the rows below), `tenant_id`, `include[]`, `domains{}`, `dry_run` |
| `scan_sca` | Run SCA/SBOM scan — see Step 2E | `tenant_id`, `repo_name`/`repository`, `mode` (QUICK/FULL) |
| `scan_secret` | Run secret scan (TruffleHog/Gitleaks) — see Step 2E | `tenant_id`, `repository`/`repoName`, `baseRef`, `scanners[]`, `paths[]` |
| `container_scan` | Scan Dockerfiles/K8s manifests + base images — see Step 2E | `repository`, `tenant_id`, `mode` (FAST/SMART/DEEP) |
| `container_taint_paths` | Taint paths through Dockerfile data flow | `repository`, `tenant_id`, `source_type`, `sink_type` |
| `container_base_images` | Base images across Dockerfiles with CVE counts + freshness | `repository`, `tenant_id` |
| `iac_scan` | Trigger an IaC scan (Trivy/KICS/Checkov) — see Step 2E | `tenant_id`, `repo_name`, `scanMode`, `paths[]` |
| `iac_findings` | List IaC findings | `tenant_id`, `repo_name`, `severity`, `ruleIdPrefix` |
| `iac_assets` | List IaC cloud assets extracted from a repo | `tenant_id`, `repo_name`, `provider`, `framework` |
| `iac_controls` | List detected IaC compensating controls | `tenant_id`, `repo_name` |
| `iac_taint_trace` | Trace taint through IaC variable chains | `tenant_id`, `repo_name` |
| `iac_toxic_combos` | Evaluate toxic combinations (`PHX-IAC-COMB-*`) | `tenant_id`, `repo_name` |
| `iac_module_check` | Check a Terraform module source for supply-chain risk | `tenant_id`, `repo_name`, `moduleSource` |
| `zero_day_scan` | Analyze commits for pre-CVE vulnerability patches — see Step 2E | `repoPath`, `mode` (PR_DIFF/RECENT_COMMITS/FULL_REPO), `workspaceId`, `tenant_id` |
| `zero_day_findings` | List 0-day findings for a workspace | `tenant_id`, `workspaceId` |
| `zero_day_monitor_create` | Create a persistent 0-day monitor for a GitHub repo | `repoUrl`, `tenant_id`, `workspaceId` |
| `zero_day_monitor_list` | List 0-day monitors for a workspace | `tenant_id`, `workspaceId` |
| `dast_template_list` | Browse DAST detection templates (Nuclei/ZAP) | `tenant_id`, `engine`, `severity`, `tag`, `cve` |
| `dast_template_bundles` | List named DAST template bundles/profiles | `tenant_id`, `engine` |
| `dast_template_resolve` | Resolve a bundle/selector to a template count + preview | `tenant_id`, `bundle` or `selector` |
| `dast_scan` (flag-gated) | Launch a live external DAST assessment against a registered, ownership-verified target — see Step 2D | `tenant_id`, `target_id`, `scan_mode` |
| `run_assessment` | Full OWASP/ASVS security assessment — compat alias, resolves `repoName` for you — see Step 2E | `repoName`, `tenant_id`, `scope` (FULL/PATH), `targetPath`, `depth` (LIGHT/STANDARD/DEEP) |
| `security_assessment` | Same engine as `run_assessment`, called with an already-resolved repo path — prefer `run_assessment` unless you have one | `repoPath` (**not** `repoName`), `tenant_id`, `scope`, `targetPath`, `depth`, `categories[]` |
| `credential_blast_radius` | Credential exposure analysis | `repoName` |
| `refresh_vulnerability_stats` | Refresh vuln stats | `repoName` |
| `scaffold_*` | Generate agent scaffolding | varies |
| `resolve_workspace_repo` | Resolve workspace→repo | `workspaceId` or `repoName` |
| `compatibility_contract` | Check client/server compatibility | — |

> Graph-navigation tools (`analyze`, `query`, `context`, `impact`, `entry_points`, `key_functions`,
> `communities`, `contributors`, `processes`, `overview`) are on the **`phoenix-graph`** endpoint.
> Connect `phoenix-graph` or use the `purple-graph-navigator` skill for those.

> The tools listed above are the ones this skill exercises. The full set exposed by
> `phoenix-security` is feature-gated per deployment and enumerated at runtime via `tools/list` —
> do not assume a fixed total tool count.

**Feature gates:**
- Hunt tools hidden when `phx.ctfhunt.api-enabled=false`
- Prometheus tools absent when `features.prometheus=false`
- `dast_scan` absent from `tools/list` unless BOTH `phx.features.external-assessment` and
  `phx.dast.mcp-scan-enabled` are true (both default off); even when advertised, invoking it also
  requires the caller org's `webApiDast` entitlement and a target with a `VERIFIED` host
  authorization

---

## Error handling

| Error | Cause | Resolution |
|-------|-------|------------|
| Tool missing from `tools/list` | Feature flag off, wrong endpoint, or graph tool called on security endpoint | Check feature flag; graph tools only exist on `phoenix-graph` |
| `tenant_id required` (Prometheus MCP) | Missing arg | Pass dev user ID for local; JWT subject for prod |
| `401 Unauthorized` (REST), empty body | **Wrong credential type** — a `phx_live_` API key is being sent where a `phx_at_` access token is required. `/api/v1/external/**` rejects any other prefix before looking it up, so there is no body and no server log line | Exchange it: `curl -s -X POST -H "Authorization: Bearer $PHX_MCP_TOKEN" -H 'Content-Type: application/json' -d '{}' "$PHX_BASE_URL/api/v1/external/auth/token" | jq -r .accessToken` |
| `401 Unauthorized` (REST) | Token expired | Re-run `scripts/purple auth token` (auto-refreshes); if that fails, run `scripts/purple auth login` |
| `403` on SSE stream (REST) | Missing `stream_secret` | Re-read from start-run response |
| `graphStale: true` | Graph outdated | Connect `phoenix-graph` and call `analyze` (MCP) or `POST /api/analyze/start` (REST) |
| Hunt `FAILED` | Budget or worker error | Reduce `max_targets`, increase `budget_usd`, check `failureReason` |
| Prometheus `429` | Concurrency cap (5/tenant) | Wait for `Retry-After` |
