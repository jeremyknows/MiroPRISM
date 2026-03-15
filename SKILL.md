---
name: miroprism
description: |
  MiroPRISM — Adversarial two-round review protocol. Extends PRISM with a mandatory
  second round where every reviewer must respond to all R1 findings with evidence
  requirements enforced by a structured anti-herding guardrail. Eliminates cascade
  sycophancy: reviewers cannot agree with a finding without independent evidence,
  cannot change their verdict without citing cause, and can mark findings UNCERTAIN
  rather than force a weak call. Core insight: A finding that survives explicit
  challenge is more reliable than one that was never challenged.
license: MIT
compatibility: Works with any agent that can spawn subagents
metadata:
  author: jeremyknows
  version: "1.1.0"
---

# MiroPRISM v1 — Adversarial Two-Round Review Protocol

Two-round review protocol that eliminates cascade sycophancy — the pattern where early findings anchor later reviewers' opinions, producing false consensus — through structured evidence-gated disagreement.

**What is PRISM?** PRISM runs multiple specialist reviewers in parallel, each seeing only the artifact and not each other's findings (blind, isolated). MiroPRISM extends this with a mandatory second round: every reviewer sees all Round 1 findings and must explicitly AGREE, DISAGREE, or mark UNCERTAIN — with independent evidence required for each stance. The result: findings are labeled by whether they survived challenge, not just whether reviewers agreed.

## Core Principles

> "A finding that survives explicit challenge is more reliable than one never challenged."

> "UNCERTAIN is a valid and preferred stance over weak agreement."

> "Unchallenged ≠ correct. Challenged and surviving = high confidence."

**Key differences from standard review protocols:**
- R1: 5 specialist reviewers analyze independently and in isolation — each sees only the artifact
- Phase 2: All R1 findings are sanitized, anonymized, and broadcast as a shared digest
- R2: Every reviewer responds to every finding with AGREE / DISAGREE / UNCERTAIN + evidence
- Synthesis labels confidence based on whether findings were challenged and survived — not just consensus
- Prompt injection is blocked at the digest layer via structured finding templates

---

## How to Invoke MiroPRISM

| Mode | Say This | Reviewers | Rounds | Model | Est. Cost |
|------|----------|-----------|--------|-------|-----------|
| **Standard** | "MiroPRISM this" / "Run MiroPRISM" | 5 (all) | 2 | Sonnet | ~$0.70 |
| **Budget** | "Budget MiroPRISM" | 3 (Sec + DA + Integration) | 2 | Haiku | ~$0.08 |
| **Extended** | "MiroPRISM this, max 3 rounds" | 5 (all) | 2–3 | Sonnet | ~$1.20 |

**Budget reviewer override:** "Budget MiroPRISM with Performance" → Security + DA + Performance

**Flags:**
- `--review-digest` — pause before R2 to surface the digest log for manual approval

**Examples:**
```
"MiroPRISM this design doc"
"Budget MiroPRISM on the auth flow"
"MiroPRISM this, max 3 rounds"
"MiroPRISM this, review digest log before R2"
"Budget MiroPRISM with Simplicity"
```

---

## When to Use MiroPRISM vs PRISM

| Situation | Use |
|-----------|-----|
| Architecture decisions, major forks, things living 6+ months | MiroPRISM |
| Security-sensitive changes, open source releases | MiroPRISM |
| High-stakes decisions where consensus drift is a real risk | MiroPRISM |
| Bug fixes, minor refactors, reversible decisions | PRISM |
| Fast checks, urgent reviews | PRISM (or Budget PRISM) |
| You've already run PRISM once and want a deeper pass | MiroPRISM |

---

## Evidence Rules

All reviewers must follow these rules. Included verbatim in every reviewer prompt.

```
EVIDENCE RULES (mandatory for all MiroPRISM reviewers):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.
```

---

## The MiroPRISM Flow — Complete Orchestrator Checklist

Follow these steps exactly.

---

### Phase 1 — Independent Review (R1)

**Step 1: Generate slug**

Derive kebab-case slug from the review subject:
```
"MiroPRISM design v4" → miroprism-design-v4
"Auth flow refactor" → auth-flow-refactor
```
Sanitize: lowercase, alphanumeric + hyphens only, max 60 chars. No path separators.

**Collision handling:** If `analysis/miroprism/runs/<slug>/` already exists with mtime <1hr:
- Append suffix: `<slug>-2`, `<slug>-3`, etc.
- Use the first available suffix

**Working directory:** All paths in this skill are relative to your workspace root.
- OpenClaw: `~/.openclaw/agents/main/workspace`
- Other agents: your project root

**Step 2: Write lock file**
```bash
mkdir -p analysis/miroprism/runs/<slug>/r1-outputs/
touch analysis/miroprism/runs/<slug>/.lock
```
The `.lock` file prevents concurrent runs from corrupting state. Remove it only on clean completion (Step 11). If `.lock` already exists when you start, a prior run may have aborted — check before overwriting.

**Step 3: Spawn R1 reviewers in parallel**

Spawn all 5 reviewers simultaneously. Use PRISM reviewer prompts verbatim — MiroPRISM adds nothing to R1 except the output file target. Do NOT include a Prior Findings Brief in R1 (reviewers are fully blind, same as PRISM).

Each reviewer writes their output to:
```
analysis/miroprism/runs/<slug>/r1-outputs/<role>.md
```
Where `<role>` is: `security`, `performance`, `simplicity`, `integration`, `da`

**Step 4: Wait for R1 completion**

Wait up to 10 minutes. Require 4/5 complete before proceeding.

**Timeout handling:** If a reviewer doesn't complete:
- Write stub file: `--- TIMEOUT: reviewer did not complete within 10m ---` (no role name — preserves identity-stripping in Phase 2)
- Log timeout with timestamp in `R1-digest-log.md` (written in Phase 2)
- Do NOT name the timed-out role in the digest
- Flag in synthesis: *"⚠️ Incomplete Review: one reviewer timed out in R1. Second run recommended."*

---

### Phase 2 — Digest Compilation

**Step 5: Sanitize R1 outputs**

Apply all 9 sanitization rules to every R1 finding before compiling the digest:

1. Strip all quoted code blocks, verbatim excerpts, and inline code
2. Strip all URLs — replace with `[reference]`
3. Strip all JSON, structured data, SQL snippets — replace with `[structured data]`
4. Replace all stripped content with the finding template (see rule 9)
5. Do NOT group findings by verdict-leaning (grouping implies vote count)
6. Randomize finding order within the digest
7. Add header: `Round 1 Findings (randomized order, no consensus implied)`
8. Strip all reviewer identity signals (no role names, no "the security reviewer found...")
9. **Enforce finding description template — no freeform narrative:**
   ```
   [FINDING_TYPE] at [location]: [one-sentence plain-English description]
   ```
   `FINDING_TYPE` MUST be one of: `INJECTION` | `LOGIC_BUG` | `SECURITY_RISK` | `PERFORMANCE_ISSUE` | `DESIGN_CONCERN` | `INTEGRATION_GAP` | `SIMPLICITY_ISSUE`

   Example:
   ```
   SECURITY_RISK at [Phase 2 sanitization step]: Finding descriptions accept freeform narrative, creating a prompt injection surface.
   DESIGN_CONCERN at [Phase 3 guardrail]: Anti-herding rules allow verdict changes without explicit evidence requirement.
   ```

**Step 6: Write digest and transparency log**

Write sanitized digest to:
```
analysis/miroprism/runs/<slug>/r1-digest.md
```

Digest format:
```markdown
## Round 1 Findings (randomized order, no consensus implied)

1. DESIGN_CONCERN at [location]: one-sentence plain-English description
2. SECURITY_RISK at [location]: one-sentence plain-English description
[...]

## Unresolved Split Points from R1
- [topic where R1 findings diverged — no attribution, no names]
```

Write transparency log to:
```
analysis/miroprism/runs/<slug>/R1-digest-log.md
```

Transparency log contents:
- Input finding counts per reviewer (counts only, not role names)
- Sanitization counts (e.g., "3 code blocks stripped, 2 URLs replaced, 1 JSON block replaced")
- SHA256 of each R1 input file (for post-hoc verification)
- Any findings excluded or reframed, with rationale
- Timestamp of digest compilation
- Any timeouts logged here

**Step 7: If `--review-digest` flag**

Pause. Post the transparency log summary to the user. Wait for explicit approval before spawning R2 reviewers.

---

### Phase 3 — Response Round (R2)

**Step 8: Spawn R2 reviewers in parallel**

Each reviewer receives:
1. The original reviewed artifact
2. Their own R1 output
3. The sanitized `r1-digest.md`
4. The 3-rule anti-herding guardrail (copy verbatim — see below)
5. The required R2 response format (copy verbatim — see below)

Each reviewer writes to:
```
analysis/miroprism/runs/<slug>/r2-outputs/<role>.md
```

**Anti-herding guardrail (copy this EXACTLY into every R2 prompt):**
```
ROUND 2 RULES:

1. AGREE with a finding → cite your independent evidence from the reviewed artifact
   (may update your verdict)
2. DISAGREE with a finding → cite contradicting evidence or name the specific
   logical flaw in the finding's reasoning (may update your verdict)
3. UNCERTAIN on a finding → state what evidence would resolve it
   (valid and preferred over weak agreement; may update your verdict)

Silence on a finding = implicit agreement. Respond to ALL findings in the digest.
New findings triggered by R1 context are welcome — cite the R1 item that surfaced them.
```

**Required R2 response format (copy this EXACTLY into every R2 prompt):**
```
## My R1 Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]
## My R2 Verdict: [same or updated]
(Verdict changes are implicit in your AGREE/DISAGREE/UNCERTAIN responses — no separate
rationale section needed. Change your verdict only if your responses support it.)

## Responses to R1 Digest

### Agreements (cite your independent evidence from the artifact)
- [finding from digest] → AGREE — [your independent evidence]

### Disagreements (cite contradicting evidence or name the logical flaw)
- [finding from digest] → DISAGREE — [contradicting evidence / specific flaw]

### Uncertainties (insufficient evidence either way)
- [finding from digest] → UNCERTAIN — [what evidence would resolve this]

## New Findings (triggered by R1 context)
- [finding] — [citation from artifact] — triggered by: [R1 digest item that surfaced it]
```

**Step 9: Validate R2 responses**

Before synthesis, run these 4 validation checks on each R2 output:

1. **Structural:** All required sections present — if missing, note in synthesis as INCOMPLETE
2. **Verdict drift:** If R2 Verdict ≠ R1 Verdict AND the AGREE/DISAGREE/UNCERTAIN responses don't support the change → flag `[FLAGGED: verdict change unsubstantiated]` in synthesis
3. **Citation validity:** References to findings that don't exist in the digest → mark `[UNVALIDATED]` (still included, lower weight)
4. **Evidence depth:** <50 chars of evidence in an AGREE or DISAGREE response → log as `[LOW-CONFIDENCE]` in synthesis (not rejected)

All validation flags written to `R1-digest-log.md`.

**Step 10: UNCERTAIN rate check**

Count all R2 responses across all reviewers. If >75% are marked UNCERTAIN:

> ⚠️ **High UNCERTAIN rate detected (>75%).** This may indicate genuine ambiguity or review dilution. Recommend re-running with fresh reviewers or manual review before acting on synthesis.

Post this warning before proceeding to synthesis.

---

### Phase 4 — Synthesis

**Step 11: Synthesize**

Use the synthesis template below. Remove `.lock` file after writing the archive.

Write synthesis to:
```
analysis/miroprism/archive/<slug>/YYYY-MM-DD-review-N.md
```
(N increments if the slug has been reviewed before)

---

## Reviewer Roles

Identical to PRISM Standard Mode. R1 prompts are PRISM prompts verbatim.

| Reviewer | Focus | Key Question |
|----------|-------|--------------|
| 🔒 **Security Auditor** | Attack vectors, trust boundaries | "How could this be exploited?" |
| ⚡ **Performance Analyst** | Metrics, benchmarks, overhead | "Show me the numbers" |
| 🎯 **Simplicity Advocate** | Complexity reduction | "What can we remove?" |
| 🔧 **Integration Engineer** | Compatibility, migration gaps | "How does this fit?" |
| 😈 **Devil's Advocate** | Assumptions, risks, regrets | "What are we missing?" |

**Budget Mode (3 reviewers):** Security Auditor + Devil's Advocate + Integration Engineer (default) or override with "Budget MiroPRISM with [role]".

**R1 prompts:** Use PRISM SKILL.md reviewer prompts verbatim — Security, Performance, Simplicity, Integration, DA sections. MiroPRISM adds nothing to R1. DA in R1 is still blind (no Prior Findings Brief), same as PRISM.

---

## R1 Reviewer Prompts

MiroPRISM R1 is identical to a standard PRISM review — same 5 reviewer roles, same prompts, same evidence rules. The only MiroPRISM-specific addition is the output file target appended to each prompt.

**Prompts last synced from PRISM v2.0.1.** If you have PRISM installed, you can also use its prompts directly.

Append this line to every R1 reviewer prompt before sending:
```
Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/<role>.md
Include all findings, citations, and your final verdict in that file.
```

### Security Auditor (R1)

```
You are the Security Auditor in a MiroPRISM Round 1 review.

Focus: Trust boundaries, attack vectors, data exposure.

EVIDENCE RULES (mandatory):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

Your job: Find security issues — trust boundary violations, attack vectors,
data exposure risks, secrets handling problems.

Questions to answer:
1. What are the top 3 ways this could be exploited? (cite specific code/config)
2. What security guarantees are we losing vs gaining?
3. What assumptions about trust might be wrong?

Output format:
- Risk Assessment: [High/Medium/Low]
- New Attack Vectors: [numbered list with severity, file citations, and fixes]
- Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]

Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/security.md
```

### Performance Analyst (R1)

```
You are the Performance Analyst in a MiroPRISM Round 1 review.

Focus: Measurable metrics, not vibes. Numbers beat intuition.

EVIDENCE RULES (mandatory):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

Your job: Find performance issues with specific measurements.

Questions to answer:
1. What's the latency/memory/token/cost impact? (specific numbers)
2. Are there benchmarks we can reference or measure?
3. What's the performance worst-case scenario?

Output format:
- Metrics: [specific numbers with units]
- Comparison: [before vs after, with measurements]
- New Risks: [with citations and fixes]
- Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]

Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/performance.md
```

### Simplicity Advocate (R1)

```
You are the Simplicity Advocate in a MiroPRISM Round 1 review.

Focus: Complexity reduction. Challenge every added component.

EVIDENCE RULES (mandatory):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

Your job: Find what can be removed or simplified.

Questions to answer:
1. What can we remove without losing core value?
2. Is this the simplest solution that works?
3. What "nice-to-haves" are disguised as requirements?

Output format:
- Complexity Assessment: [count of components, dependencies, moving parts]
- Essential vs Cuttable: [two lists with specific citations]
- Simplification Opportunities: [with specific file paths and changes]
- Verdict: [APPROVE | APPROVE WITH CONDITIONS | SIMPLIFY FURTHER | REJECT]

Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/simplicity.md
```

### Integration Engineer (R1)

```
You are the Integration Engineer in a MiroPRISM Round 1 review.

Focus: How this fits the existing system. Migration and compatibility.

EVIDENCE RULES (mandatory):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

Your job: Find integration risks, breaking changes, and migration gaps.

Questions to answer:
1. What's the migration path for existing users?
2. What breaks if we deploy this?
3. How long until this is stable?

Output format:
- Integration Effort: [estimate with breakdown]
- Breaking Changes: [list with file citations]
- Migration Strategy: [phased steps]
- Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]

Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/integration.md
```

### Devil's Advocate (R1)

```
You are the Devil's Advocate in a MiroPRISM Round 1 review.

Your job: Find the flaws. Challenge assumptions. Be ruthlessly skeptical.
When you approve with no conditions, something is probably wrong.

EVIDENCE RULES (mandatory):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

IMPORTANT: You review with fresh eyes, independently. Do NOT look at what
other reviewers found. Your independence is what makes your perspective valuable.

Questions to answer:
1. What assumptions underpin this that might not hold?
2. What will we regret in 6 months?
3. What's the strongest argument AGAINST this decision?
4. What are we not seeing?

Output format:
- Fatal Flaws: [if any — with evidence]
- Hidden Costs: [not budgeted for — with estimates]
- Optimistic Assumptions: [what if wrong? — cite specific claims]
- 6-Month Regrets: [what we'll wish we'd kept]
- Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]

Write your complete review output to:
analysis/miroprism/runs/<slug>/r1-outputs/da.md
```

---

## R2 Reviewer Prompt Template

**Assembly instructions:** Replace all `[INSERT ...]` placeholders with actual content before sending to a reviewer. These are slots for the orchestrator to fill — not template strings to leave as-is.

Assemble this prompt for each R2 reviewer:

```
You are the [ROLE NAME] in Round 2 of a MiroPRISM review.

In Round 1, you reviewed [ARTIFACT DESCRIPTION] and produced the findings below.
You are now receiving a sanitized digest of ALL Round 1 findings from all reviewers.

Your task: respond to every finding in the digest with AGREE, DISAGREE, or UNCERTAIN.
Evidence is required for each stance.

---
## The Artifact Being Reviewed

[INSERT ORIGINAL ARTIFACT / LINK TO ARTIFACT]

---
## Your Round 1 Output

[INSERT THIS REVIEWER'S r1-outputs/<role>.md VERBATIM]

---
## Round 1 Digest (all reviewers, sanitized, randomized)

[INSERT r1-digest.md VERBATIM]

---
EVIDENCE RULES (mandatory for all MiroPRISM reviewers):
1. Before analyzing, read at least 3 specific files relevant to your focus.
2. Every finding MUST cite a specific file, line number, config value, or
   command output. Quote directly from what you read.
3. Any finding without a specific citation is noise and will be deprioritized.
4. Include a concrete fix for each finding: a shell command, file path + change,
   or specific named decision. "Consider improving" is not acceptable.

---
ROUND 2 RULES:

1. AGREE with a finding → cite your independent evidence from the reviewed artifact
   (may update your verdict)
2. DISAGREE with a finding → cite contradicting evidence or name the specific
   logical flaw in the finding's reasoning (may update your verdict)
3. UNCERTAIN on a finding → state what evidence would resolve it
   (valid and preferred over weak agreement; may update your verdict)

Silence on a finding = implicit agreement. Respond to ALL findings in the digest.
New findings triggered by R1 context are welcome — cite the R1 item that surfaced them.

---
## My R1 Verdict: [APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]
## My R2 Verdict: [same or updated]
(Verdict changes are implicit in your AGREE/DISAGREE/UNCERTAIN responses — no separate
rationale section needed. Change your verdict only if your responses support it.)

## Responses to R1 Digest

### Agreements (cite your independent evidence from the artifact)
- [finding from digest] → AGREE — [your independent evidence]

### Disagreements (cite contradicting evidence or name the logical flaw)
- [finding from digest] → DISAGREE — [contradicting evidence / specific flaw]

### Uncertainties (insufficient evidence either way)
- [finding from digest] → UNCERTAIN — [what evidence would resolve this]

## New Findings (triggered by R1 context)
- [finding] — [citation from artifact] — triggered by: [R1 digest item that surfaced it]
```

Write your complete R2 output to:
```
analysis/miroprism/runs/<slug>/r2-outputs/<role>.md
```

---

## Synthesis Template

```markdown
## MiroPRISM Synthesis — [Slug]

**Review #:** [nth review of this topic]
**R1 Reviewers:** [list with R1 verdicts]
**R2 Reviewers:** [list with R2 verdicts]
**Verdict change rate:** [X/5 reviewers changed verdict R1→R2]
[If any reviewer timed out: "⚠️ One reviewer timed out in R1 — partial synthesis. Second run recommended."]
[If UNCERTAIN rate >75%: "⚠️ High UNCERTAIN rate — synthesis confidence reduced."]

---

### Findings

All findings from R1 and R2, tiered by confidence. Each finding gets an inline label:

- `[HIGH]` — challenged in R2 with DISAGREE, held with counter-evidence. Act on these.
- `[STANDARD: VALIDATION REQUIRED]` — not challenged in R2. Unchallengeable ≠ correct. Verify independently before acting.
- `[STANDARD: IMPLICIT AGREE]` — reviewer was silent (no explicit AGREE/DISAGREE/UNCERTAIN response). Counts as agreement for synthesis but carries no evidence weight. Do NOT conflate with active agreement.
- `[FLAGGED: CONSENSUS DRIFT]` — verdict changed in R2 without substantiated independent evidence.
- `[FLAGGED: UNVALIDATED CITATION]` — reviewer cited a finding not present in the digest.
- `[LOW-CONFIDENCE]` — AGREE/DISAGREE response with <50 chars of evidence.

Present [HIGH] findings first, then [STANDARD], then [FLAGGED].

---

### Unresolved Disagreements

Findings where at least one reviewer marked DISAGREE in R2 with citations, AND at least one
other reviewer marked AGREE or held their R1 finding with counter-evidence. These are the most
valuable outputs — genuine expert disagreement on the same evidence base.

Format per disagreement:
**[Finding description]**
- For: [summary of AGREE evidence]
- Against: [summary of DISAGREE evidence]
- Recommended action: [specific resolution path]

If no unresolved disagreements: "No unresolved disagreements — all challenged findings reached R2 resolution."

---

### Consensus Points

Findings that went unchallenged OR where all R2 responses were AGREE with independent evidence.
Note: these carry STANDARD confidence, not HIGH — they were not tested under challenge.

---

### VALIDATION REQUIRED — Executive Summary

The following findings were not challenged in R2. They require independent verification before acting:
[List STANDARD findings with brief description]

These may be correct. They are not confirmed under adversarial conditions.

---

### Final Verdict

[APPROVE | APPROVE WITH CONDITIONS | NEEDS WORK | REJECT]
Confidence: [percentage]

Rationale: [1–2 sentences — what drove the verdict]

### Conditions (if AWC or NEEDS WORK)
[Numbered list — specific, actionable, with file paths or commands]
```

---

## Extended Mode (max 3 rounds)

After R2 synthesis is complete, measure the R2 delta:
- Count new findings in R2 that were NOT present in R1 (items under "New Findings" sections)
- Delta = (new R2 findings) / (total R1 findings). Example: R1 had 10 findings, R2 added 3 new → delta = 30% → trigger R3.
- If delta > 20%: spawn R3 using the same broadcast protocol
  - R3 digest = combined R1 + R2 findings, re-sanitized and re-randomized
  - R3 uses identical guardrail and response format
  - Note in synthesis: *"R3 triggered: R2 delta was [X]% (>20% threshold)"*
- If delta ≤ 20%: stop at R2, note in synthesis: *"R3 skipped: R2 delta [X]% — diminishing returns threshold not met"*

Hard cap: `max_rounds = 3`. No R4.

---

## File Structure

```
analysis/miroprism/
  runs/
    <slug>/
      .lock                     # written on start, removed on clean completion
      r1-outputs/
        security.md
        performance.md
        simplicity.md
        integration.md
        da.md
      R1-digest-log.md          # transparency log (written before R2)
      r1-digest.md              # sanitized, randomized digest → all R2 reviewers
      r2-outputs/
        security.md
        performance.md
        simplicity.md
        integration.md
        da.md
  archive/
    <slug>/
      YYYY-MM-DD-review-1.md    # final synthesis, N increments per run
      YYYY-MM-DD-review-2.md
```

---

## Verdict Scale

| Verdict | Meaning | When to Use |
|---------|---------|-------------|
| **APPROVE** | No blocking issues after two rounds | Clean bill of health |
| **APPROVE WITH CONDITIONS** | Issues found, none blocking | Ship it, fix these soon |
| **NEEDS WORK** | Blocking issues found, fixable | Don't ship until resolved |
| **REJECT** | Critical issues or fundamental design problems | Requires rethink |

---

## Post-Launch Validation Checklist

Track these metrics across the first 10 real MiroPRISM runs.

**After every run, append one row to `analysis/miroprism/metrics.tsv`:**
```
date	slug	verdict_change_rate	uncertain_rate	r1_finding_count	r2_new_finding_count	high_finding_count	unresolved_disagreement_count
```

Example row:
```
2026-03-15	auth-flow-refactor	0.40	0.12	10	3	4	2
```

Create the file on first run:
```bash
echo "date\tslug\tverdict_change_rate\tuncertain_rate\tr1_finding_count\tr2_new_finding_count\thigh_finding_count\tunresolved_disagreement_count" > analysis/miroprism/metrics.tsv
```

**Metrics to track:**

1. **R1→R2 finding delta** — `r2_new_finding_count / r1_finding_count`. If consistently <10%, consider dropping to 1 round.
2. **Verdict change rate** — `verdict_change_rate`: % of reviewers changing verdict R1→R2. If >60%, revisit broadcast vs selective pairing.
3. **UNCERTAIN usage rate** — `uncertain_rate`: are reviewers using UNCERTAIN genuinely? Watch for 0% (too confident) and >50% (avoidance).
4. **Budget vs Standard quality gap** — Does Budget (3 reviewers) miss HIGH findings that Standard catches? Compare `high_finding_count` across modes.
5. **Reviewer count sensitivity** — On 2 runs, use 7 reviewers instead of 5. Does `high_finding_count` change?
6. **Injection resistance** — On adversarial content, does the structured finding template hold? Check `R1-digest-log.md` sanitization counts.

**Decisions after 10 runs:** round cap, reviewer count default, broadcast vs selective pairing, Budget default domain specialist.

To review your data:
```bash
# View all runs
cat analysis/miroprism/metrics.tsv | column -t -s $'\t'

# Average verdict change rate
awk -F'\t' 'NR>1 {sum+=$3; count++} END {print "Avg verdict change rate:", sum/count}' analysis/miroprism/metrics.tsv
```

---

## Cost Reference

| Variant | Reviewers | Rounds | Model | Tokens | Est. Cost |
|---------|-----------|--------|-------|--------|-----------|
| Standard | 5 | 2 | Sonnet | ~97K | ~$0.50–0.80 |
| Budget | 3 | 2 | Haiku | ~40K | ~$0.08 |
| Extended | 5 | 2–3 | Sonnet | ~145K | ~$0.75–1.20 |

Token breakdown (Standard):
- R1 × 5: ~35K (~5K in, ~2K out each)
- Phase 2 digest: ~500 (orchestrator)
- R2 × 5: ~42.5K (~7K in, ~1.5K out each)
- Synthesis: ~7K

**⚠️ Large artifact warning:** R2 sends the original artifact to every reviewer. If your artifact is >5K tokens (~4K words / ~20KB), multiply R2 cost by the number of reviewers. A 20K-token design doc adds ~100K tokens to Standard R2 alone — pushing total cost to ~$1.50–2.00.

For large artifacts, use one of these strategies:
- **Store externally:** Reference by file path or URL in R2 instead of pasting verbatim. Saves 15–25% on R2.
- **Use Budget mode:** 3 reviewers instead of 5 cuts large-artifact R2 cost by 40%.
- **Truncate context:** If the artifact has clearly irrelevant sections, trim before invoking.

---

## Anti-Patterns

**Don't:**
- ❌ Let R1 reviewers see each other's findings (that's what Phase 2 is for, with sanitization)
- ❌ Send freeform finding descriptions in the digest (bypasses injection defense)
- ❌ Accept verdict changes without checking AGREE/DISAGREE support
- ❌ Treat VALIDATION REQUIRED findings as confirmed — they weren't tested under challenge
- ❌ Skip the .lock file — concurrent runs will corrupt state

**Do:**
- ✅ Enforce the structured finding template at Phase 2 — reject freeform descriptions
- ✅ Check UNCERTAIN rate before synthesis — >75% is a signal, not a verdict
- ✅ Surface Unresolved Disagreements prominently — they're the most valuable output
- ✅ Archive every synthesis — future runs can compare delta across reviews
- ✅ Remove .lock on clean completion; leave it if the run aborts (signals dirty state)
