# MiroPRISM — Adversarial Two-Round Review Protocol

> A finding that survives explicit challenge is more reliable than one that was never challenged.

MiroPRISM is a standalone fork of [PRISM](https://github.com/jeremyknows/prism) that adds a mandatory second debate round. Where PRISM runs reviewers in isolation and merges at the end, MiroPRISM broadcasts a sanitized digest of all R1 findings and requires every reviewer to respond with AGREE, DISAGREE, or UNCERTAIN — with independent evidence required for each stance.

**The problem it solves:** PRISM reviewers converge toward consensus without genuine disagreement. The first voice anchors everything. MiroPRISM breaks that pattern by forcing explicit engagement with opposing findings before synthesis.

## What it does

- **R1** — 5 specialist reviewers run in complete isolation (identical to PRISM)
- **Phase 2** — Orchestrator sanitizes all R1 findings (strips code, URLs, JSON; enforces structured templates to block prompt injection) and broadcasts a randomized digest
- **R2** — Every reviewer responds to every finding: AGREE (cite evidence), DISAGREE (cite contradiction), or UNCERTAIN (state what would resolve it)
- **Synthesis** — Findings labeled `[HIGH]` (challenged + survived), `[STANDARD: VALIDATION REQUIRED]` (unchallenged), or `[FLAGGED]` (drift, unvalidated citation). Unresolved Disagreements surfaced as the primary output.

## Install

**Claude Code / OpenClaw:**
```bash
git clone https://github.com/jeremyknows/miroprism ~/.openclaw/skills/miroprism
```

**Cursor / Windsurf / other:**
```bash
git clone https://github.com/jeremyknows/miroprism /path/to/your/skills/miroprism
```

Then restart your agent or reload skills.

## Usage

Just say it — no configuration needed:

```
"MiroPRISM this"                              → Standard (5 reviewers, 2 rounds, Sonnet)
"Budget MiroPRISM"                            → 3 reviewers, 2 rounds, Haiku (~$0.08)
"Budget MiroPRISM with Performance"           → Security + DA + Performance
"MiroPRISM this, max 3 rounds"                → Auto-triggers R3 if R2 delta >20%
"MiroPRISM this, review digest log before R2" → Pause for manual digest approval
```

## When to use MiroPRISM vs PRISM

| Situation | Use |
|-----------|-----|
| Architecture decisions, major forks, things living 6+ months | MiroPRISM |
| Security-sensitive changes, open source releases | MiroPRISM |
| High-stakes decisions where consensus drift is a real risk | MiroPRISM |
| Bug fixes, minor refactors, reversible decisions | PRISM |
| Fast checks, urgent reviews | PRISM (or Budget PRISM) |

## Variants

| Variant | Reviewers | Rounds | Model | Est. Cost |
|---------|-----------|--------|-------|-----------|
| Standard | 5 | 2 | Sonnet | ~$0.50–0.80 |
| Budget | 3 | 2 | Haiku | ~$0.08 |
| Extended | 5 | 2–3 | Sonnet | ~$0.75–1.20 |

## File structure

```
analysis/miroprism/
  runs/<slug>/
    .lock               # concurrent run guard
    r1-outputs/         # blind R1 reviewer outputs
    R1-digest-log.md    # transparency log (SHA256, sanitization counts)
    r1-digest.md        # sanitized, randomized digest sent to R2
    r2-outputs/         # R2 response outputs
  archive/<slug>/
    YYYY-MM-DD-review-N.md   # final synthesis
```

## Security

MiroPRISM's digest pipeline sanitizes R1 findings before broadcast:
- Strips all code blocks, URLs, JSON, and structured data
- Enforces a structured finding template with allowlisted `FINDING_TYPE` values — no freeform narrative
- Randomizes finding order to remove implicit vote-count signal
- Strips all reviewer identity signals

The transparency log (`R1-digest-log.md`) records SHA256 of every R1 input and sanitization counts for post-hoc verification.

## Limitations

- Requires an agent capable of spawning 5+ parallel subagents
- R2 token cost is ~2.5–3x PRISM — not appropriate for quick checks
- Reviewer count (5) is a starting point, not a validated optimum — see Post-Launch Validation Checklist in SKILL.md
- Extended mode (3 rounds) adds cost; only use when R2 delta is high

## Relationship to PRISM

MiroPRISM is a **standalone fork**, not a PRISM extension. The execution model (feedback loop) is fundamentally different. R1 reviewer prompts are identical to PRISM; everything else is new. Do not try to bolt MiroPRISM's R2 onto an existing PRISM run.

## License

MIT — see [LICENSE.txt](LICENSE.txt)
