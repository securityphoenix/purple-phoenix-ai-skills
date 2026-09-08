# Releasing

How a new version of the Purple skills reaches customers. Three publishing options, in order of
how most people will get it.

## Option 1 — the marketplace (default, no release step)

`claude plugin marketplace add securityphoenix/purple-skills` tracks the **default branch**. A
merge to `main` is a release: the next time a customer runs
`claude plugin marketplace update phoenix-purple`, they get it.

This is why `main` must always be installable. The `validate` workflow enforces that on every push
and pull request.

## Option 2 — a pinned tag

Customers who want a fixed version can pin one. Cut a tag when the changelog gets a new version:

```bash
git tag -a v1.1.0 -m "v1.1.0"
git push origin v1.1.0
```

Then create a GitHub Release from that tag and paste the matching `CHANGELOG.md` section into the
release notes.

## Option 3 — a tarball

For an air-gapped install, the GitHub Release page's source archive is enough. The skills are
plain Markdown and the MCP configs are plain JSON/TOML — there is nothing to build.

---

## Cutting a version

Three files carry the version number. They must agree, or the marketplace advertises one version
and installs another.

1. `.claude-plugin/marketplace.json` → `plugins[0].version`
2. `plugins/purple-security/.claude-plugin/plugin.json` → `version`
3. `CHANGELOG.md` → a new `## [X.Y.Z] — YYYY-MM-DD` section at the top

Then validate, commit, merge, tag:

```bash
claude plugin validate .
claude plugin validate ./plugins/purple-security
```

### Which number to bump

| Bump | When |
|---|---|
| **Major** | A skill is removed or renamed, or an install step changes in a way that breaks existing setups |
| **Minor** | A new skill, a new scan domain, or a new capability in an existing skill |
| **Patch** | Corrections, clarifications, troubleshooting entries, MCP config fixes |

## Before you publish anything

- [ ] Both `claude plugin validate` calls pass.
- [ ] No credential in any file. Every MCP config reads its token from the environment or prompts
      for it — never hardcodes one.
- [ ] No internal path, ticket id, or planning reference in a `SKILL.md`. This repository is
      public.
- [ ] The `CHANGELOG.md` entry describes what changed **for the user**, not how it was built.
- [ ] A claim about a capability is only made if that capability actually works today. Say plainly
      when something is not yet enabled.
