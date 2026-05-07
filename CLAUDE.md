# agent-skills (personal fork: `hao-github-0216/agent-skills`)

Forked from `addyosmani/agent-skills`. Adapted for the fork owner's domain — **adversarial machine-learning research on a multi-device + SLURM/HPC pipeline**, NOT generic SaaS / web-app development. When a default skill assumes a web codebase (Core Web Vitals, browser DOM, production deploys, REST APIs), prefer the local interpretation below.

## Operating context (use this to ground every skill)

**Domain.** Adversarial ML — model robustness, data poisoning, training-time attacks/defences. Code is research-grade Python (PyTorch, NumPy, CUDA), not production software. There is no end-user, no UI, no uptime SLO; the "users" are the experiments themselves.

**Devices.** Three machines, each with a distinct role. Skills should pick the right one before suggesting an action.
- **`howardserver`** — control plane. Linux, RTX 3060 Ti 8GB. Used for code edits, repo ops, sanity-check runs (≤30 min, ≤8GB VRAM, Phase 0 quickstart only). Hosts `~/vault/`, `~/Bridge2HPC/`, `~/mcp-memory/`.
- **OSU HPC** — heavy compute. SLURM partitions: `ampere` (A100, default for Phase 1+), `dgxh` (H100, escalation only), never `dgxh200`. Submit goes through `flip` jump host via `~/Bridge2HPC/hpc/submit.sh`.
- **Macbook** — viewer / companion. SSHFS to vault, Tailscale-mounted. Code editing rare; mostly Obsidian + secondary Claude Code session.

**Bridge2HPC pipeline.** Git is the **data transport**, not just version control:
1. howardserver edits → `git push` to GitHub
2. HPC `git pull` (manual or via `submit.sh`)
3. Job runs on SLURM, results `git push` from inside the job
4. howardserver `cron` (every 2 min) runs `~/Bridge2HPC/howardserver/poll.sh` → `git pull`
5. Artifacts land in `~/<repo>/local_artifacts/<tag>/`

NEVER suggest `scp` / `rsync` between howardserver and HPC for results — git is the only sanctioned channel. Commit messages thread the cross-device conversation.

**Record-keeping.** The repo intentionally contains **zero** experiment notes / decision logs / status markdown. The authoritative journal is the Obsidian vault at `~/vault/<Project>/` on howardserver:
- `Experiments/<tag>.md` = one cell per experiment run (spec + status + result link)
- `Lessons/<topic>.md` = long-form analysis, paper notes
- `_attachments/<tag>/` = plots, screenshots
Skills that say "write a decision doc" or "create an ADR" should redirect output to the vault, not commit `.md` files into the repo.

**Cross-session memory.** `mcp-memory-service` over Tailscale (`https://howardserver.tail1c1d34.ts.net/mcp/`). Tag every memory with `mind-space:projects/<repo>` AND a `cat:` tag. See `~/.claude/CLAUDE.md` for the full protocol.

**What "ship" means here.** Not a production deploy. Shipping = `bash ~/Bridge2HPC/hpc/submit.sh <project> <tag>` after sanity-check passes locally. The "launch checklist" is: vault cell exists, code committed, partition chosen, expected runtime estimated, reproducibility seeds set.

**Hardware gotchas (already burned).** H100 (sm_90) silently NaNs on PyTorch 1.13+cu117. Pin to ampere (sm_86) or upgrade PyTorch first. This is the kind of failure mode skills like `debugging-and-error-recovery` and `source-driven-development` should know to surface.

---

This is the agent-skills project — a collection of production-grade engineering skills for AI coding agents.

## Project Structure

```
skills/       → Core skills (SKILL.md per directory)
agents/       → Reusable agent personas (code-reviewer, test-engineer, security-auditor)
hooks/        → Session lifecycle hooks
.claude/commands/ → Slash commands (/spec, /plan, /build, /test, /review, /code-simplify, /ship)
references/   → Supplementary checklists (testing, performance, security, accessibility)
docs/         → Setup guides for different tools
```

## Skills by Phase (this fork's vocabulary)

Reframed for the ML/HPC workflow described above. Web-only skills (`frontend-ui-engineering`, `browser-testing-with-devtools`) were removed — see git history of `[phase B]` commit if you need to restore.

**Define** (write the experiment spec / code spec):
- `spec-driven-development` — for ML runs the "spec" lives in `~/vault/<Project>/Experiments/<tag>.md`, not in the repo
- `idea-refine` — when the hypothesis itself is fuzzy

**Plan** (decompose into phased tasks):
- `planning-and-task-breakdown` — for ML campaigns the dependency graph is partition escalation: sanity (3060 Ti) → ampere (A100) → dgxh (H100). Each phase is a checkpoint.

**Build** (write code on howardserver, sanity-run locally):
- `incremental-implementation` — thin slices, sanity-run each before scaling up
- `context-engineering` — set up CLAUDE.md / agent context for the project
- `source-driven-development` — verify against PyTorch ↔ CUDA ↔ sm_xx matrix BEFORE suggesting an upgrade
- `api-and-interface-design` — only when defining a Python module's public surface (not REST APIs)

**Verify** (catch failure cheap, before HPC):
- `test-driven-development` — Prove-It pattern for bugs, gradient/seed checks for ML code
- `debugging-and-error-recovery` — read its ML/HPC failure-mode catalog before generic triage

**Review** (gate before submit / merge):
- `code-review-and-quality` — five-axis review
- `code-simplification` — research code drift makes this perpetually relevant
- `security-and-hardening` — low priority unless adversarial defenses involve untrusted input parsing
- `performance-optimization` — biased to web stuff upstream; for ML throughput / kernel selection, treat as a pointer not a recipe

**Ship** (submit to HPC, or push results back):
- `shipping-and-launch` — read its HPC experiment pre-launch checklist; the production-deploy variant is fallback only
- `git-workflow-and-versioning` — git is data transport here, not just version control. Bridge2HPC pattern documented inside.
- `documentation-and-adrs` — for genuine architectural decisions, but routine experiment notes go in vault, NOT as repo `.md` files
- `ci-cd-and-automation` — the local equivalent is `~/Bridge2HPC/howardserver/poll.sh` (cron-driven); not a typical CI pipeline
- `deprecation-and-migration` — relevant when retiring a project / vault folder

## Conventions

- Every skill lives in `skills/<name>/SKILL.md`
- YAML frontmatter with `name` and `description` fields
- Description starts with what the skill does (third person), followed by trigger conditions ("Use when...")
- Every skill has: Overview, When to Use, Process, Common Rationalizations, Red Flags, Verification
- References are in `references/`, not inside skill directories
- Supporting files only created when content exceeds 100 lines

## Commands

- `npm test` — Not applicable (this is a documentation project)
- Validate: Check that all SKILL.md files have valid YAML frontmatter with name and description

## Boundaries

- Always: Follow the skill-anatomy.md format for new skills
- Never: Add skills that are vague advice instead of actionable processes
- Never: Duplicate content between skills — reference other skills instead
