---
name: repo-standards
description: "Audit, standardize, or onboard repositories with organizational DevOps, AGENTS.md, CI workflows, Dependabot, and GitHub branch rulesets using a central standards repository (default: ~/.github-repo, override via STANDARDS_REPO)."
argument: "[optional: --standards-repo <path>, --repo <name>, --apply]"
---

# Repository Standards & DevOps Skill

Audits, synchronizes, and configures repositories using central repository management tooling located at `STANDARDS_REPO` (the local clone of your user/org central `.github` standards repository; defaults to `~/.github-repo`, overridable via `STANDARDS_REPO`, `--standards-repo <path>`, or caller prompt).

## Capabilities & Workflows

1. **Audit & Preview (Dry Run)**: Check whether local repositories drift from organizational standards without modifying files or opening PRs.
2. **Apply Standards (Single Repo or All Repos)**: Regenerate `AGENTS.md` (combining global base with local `PROJECT.md`), install CI workflows, Dependabot configs, CodeQL, security policies, branch protection rulesets, and create PRs.
3. **Onboard a New Repo**: Inspect framework/tier, register in `apply-standards.sh`, create initial `PROJECT.md`, and deploy baseline standards and branch protection.
4. **Update Global Standards**: Update global instructions in `templates/base-agents.md` or CI/ruleset templates, then commit/push or broadcast across repositories.
5. **Cross-Repo Command Execution**: Run arbitrary shell commands or git checks across repository tiers using `workspace-each.sh`.

---

## Central Tooling Reference

Set `STANDARDS_REPO` once (or pass `--standards-repo <path>`):

```bash
STANDARDS_REPO="${STANDARDS_REPO:-$HOME/.github-repo}"
```

All commands run against `$STANDARDS_REPO`:
* **Script**: `$STANDARDS_REPO/apply-standards.sh`
* **Multi-repo Runner**: `$STANDARDS_REPO/scripts/workspace-each.sh`
* **Templates Directory**: `$STANDARDS_REPO/templates/`
  * `base-agents.md` (Core guidelines for `AGENTS.md`)
  * `CODEOWNERS` (e.g. `* @<owner-or-team>`)
  * `SECURITY.md` (Vulnerability reporting policy)
  * `ci-flutter.yml`, `ci-node.yml` (CI PR gates)
  * `codeql.yml` (CodeQL security analysis)
  * `dependabot-flutter.yml`, `dependabot-node.yml`, `dependabot-python.yml`, `dependabot-actions-only.yml`
  * `ruleset-tier1.json` (PR required + CI pass + 1 review + linear history)
  * `ruleset-tier2.json` (PR required + branch protection)

---

## Workflows

### 1. Preview / Dry-Run Drift
To see pending drift or unapplied standards across repos:
```bash
cd "$STANDARDS_REPO"
./apply-standards.sh
```
To check a specific repo:
```bash
cd "$STANDARDS_REPO"
./apply-standards.sh --repo <repo_name>
```

### 2. Apply Standards to a Single Repo
When a single repository needs its `AGENTS.md` regenerated or its DevOps files updated:
```bash
cd "$STANDARDS_REPO"
./apply-standards.sh --apply --repo <repo_name>
```
*Note*: This checks out a branch `chore/standardize-agent-devops`, commits changes, pushes, and creates a PR if diffs exist.

### 3. Onboard a New Repository
When adding a new repository to your fleet:
1. **Detect Framework & Tier**:
   - Framework: `flutter`, `node`, `godot`, `python`, or `eleventy`.
   - Tier 1: Active production apps requiring CI + CodeQL + 1 review + linear history.
   - Tier 2: Lightweight experiments, notes, or utilities requiring baseline `AGENTS.md` + Dependabot + PR protection.
2. **Register in `$STANDARDS_REPO/apply-standards.sh`**:
   Add entry to `REPOS` array (infer `<owner>` from existing entries in `apply-standards.sh` or `gh api user --jq .login`):
   ```bash
   "<local_dir_name>|<tier>|<framework>|<has_functions:yes|no>|<owner>/<repo_name>"
   ```
3. **Scaffold `PROJECT.md` in Target Repo**:
   Ensure `<repo_path>/PROJECT.md` contains repository-specific context, tech stack notes, and run commands.
4. **Run Apply**:
   ```bash
   cd "$STANDARDS_REPO"
   ./apply-standards.sh --apply --repo <local_dir_name>
   ```

### 4. Updating Global Agent Instructions or DevOps Templates

#### A. Updating Global Agent Instructions (`templates/base-agents.md`)
When `templates/base-agents.md` is host-symlinked directly to global agent configs (e.g. `~/.claude/CLAUDE.md` and `~/.gemini/GEMINI.md`):
1. Edit `$STANDARDS_REPO/templates/base-agents.md`.
2. Commit and push in `$STANDARDS_REPO`:
   ```bash
   cd "$STANDARDS_REPO"
   git commit -am "feat: update base agent instructions"
   git push origin main
   ```
3. Changes apply immediately across symlinked local agent configs and sessions without running `apply-standards.sh` or opening PRs. (If your setup generates per-repo `AGENTS.md` files instead of symlinking, run `./apply-standards.sh --apply` to roll out updates.)

#### B. Updating DevOps Templates (CI, Dependabot, Rulesets)
For repository-level infrastructure templates:
1. Edit the relevant template in `$STANDARDS_REPO/templates/`:
   - CI workflow: `templates/ci-flutter.yml` or `templates/ci-node.yml`
   - Dependabot: `templates/dependabot-*.yml`
   - Rulesets: `templates/ruleset-tier1.json` or `templates/ruleset-tier2.json`
2. Preview changes across fleet:
   ```bash
   cd "$STANDARDS_REPO"
   ./apply-standards.sh
   ```
3. Apply changes and create PRs across repos:
   ```bash
   cd "$STANDARDS_REPO"
   ./apply-standards.sh --apply
   ```
4. Commit and push the template changes in `$STANDARDS_REPO`:
   ```bash
   cd "$STANDARDS_REPO"
   git commit -am "feat: update repo standards templates"
   git push origin main
   ```

### 5. Running Commands Across Repositories
Use `workspace-each.sh` to query or operate across all or a subset of repos:
```bash
# Check git status across Tier 1 repos
"$STANDARDS_REPO/scripts/workspace-each.sh" --tier 1 -- git status -s

# Check status across all repos
"$STANDARDS_REPO/scripts/workspace-each.sh" --all -- git status -s

# Filter by framework (e.g. Flutter repos)
"$STANDARDS_REPO/scripts/workspace-each.sh" --framework flutter -- flutter pub get
```
