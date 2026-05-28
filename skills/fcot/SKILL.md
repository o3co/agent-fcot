---
name: fcot
description: "Falsification Chain of Thought: post-hoc verification of AI judgments. Invoke with /fcot, /fcot quick, or /fcot \"judgment\"."
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

# FCoT: Falsification Chain of Thought

Post-hoc verification of your own judgments using falsificationism. You are actively trying to prove yourself wrong.

## The Core Idea

When you agree with something or make a judgment, that judgment is a hypothesis. A hypothesis is only credible if it survives attempts to falsify it. FCoT is the structured process of attempting that falsification.

**FN bias (False Negative bias):** For each counter-argument, you pre-declare a "dismissal condition" — a specific, verifiable condition under which the counter-argument is eliminated. This transforms the process from argument-from-ignorance ("I can't think of why I'm wrong, so I'm right") into self-binding logical inference ("I defined what would make me wrong, checked each one, and none held").

## When This Fires

- `/fcot` — full verification of your most recent judgment
- `/fcot "some judgment"` — full verification of the specified judgment
- `/fcot quick` — quick verification (reduced output, same core process)
- `/fcot quick "some judgment"` — quick verification of the specified judgment

### Modes

| Mode | Trigger | Counter-arguments | Output |
|------|---------|-------------------|--------|
| **full** (default) | `/fcot`, `/fcot "..."` | 3+, no upper limit | Full table + detailed verification |
| **quick** | `/fcot quick`, `/fcot quick "..."` | 1–3, prioritize highest-impact | Compact table, verification ≤ 2 sentences each |

Quick mode preserves the core mechanism (dismissal condition pre-declaration → verification) but constrains output length. Use quick as a lightweight first check; use full (default) for anything involving security, production, architecture, or irreversible decisions.

## Verification context

FCoT is structurally vulnerable to motivated reasoning when the verifying agent is the same agent that produced the judgment. Same-context verification can preserve coherence with your prior output instead of evaluating it critically (this was a confirmed finding during evaluation — see [APPROACH.md](../../APPROACH.md#context-contamination-findings)). FCoT therefore prefers a **context-stripped external verifier** by default and only falls back to in-process when an external runner is unavailable.

| Verifier | Selected when | Bias risk |
|----------|---------------|-----------|
| **external** (Codex CLI, recommended) | `codex` on `PATH` and `FCOT_VERIFIER` is not `in-process` | Low — independent process, no shared session context |
| **in-process** (fallback) | `codex` not found, or `FCOT_VERIFIER=in-process` | High — same agent that produced the judgment also falsifies it |

Selection algorithm (run once at the start of each invocation):

```bash
if [ "${FCOT_VERIFIER:-}" = "in-process" ]; then
  VERIFIER=in-process
elif [ "${FCOT_VERIFIER:-codex}" = "codex" ] && command -v codex >/dev/null 2>&1; then
  VERIFIER=codex
else
  VERIFIER=in-process   # unknown FCOT_VERIFIER value, or codex unavailable
fi
```

`FCOT_VERIFIER` contract (consumer-facing): the only meaningful values today are `codex` (try Codex, fall back to in-process if unavailable — same as omitting the variable) and `in-process` (force the fallback path). Any other value is treated as `in-process` and surfaced to the user as `Verifier: in-process (unknown FCOT_VERIFIER='<value>' — treated as fallback)`. This pre-declares the contract so future verifiers (e.g., a `claude-api` or `gemini` value) can be added without breaking existing user setups — adding a new accepted value never silently changes behavior for users who already set `FCOT_VERIFIER`. PATH-based `codex` detection only finds executables — shell aliases are not honored.

When falling back to in-process, state it explicitly in the output so the user knows: `Verifier: in-process (codex not found — output may be biased by prior reasoning).`

## Process

### Step 1: Identify the judgment

Determine what you are verifying:

1. **Argument provided** (e.g., `/fcot "Approach 1 is better"`): Find the matching judgment in your conversation history.
2. **No argument** (e.g., `/fcot`): Extract the most recent judgment, agreement, or recommendation from your last response.
3. **Nothing found**: Ask the user: "検証対象の判断が見つからない。どの判断を検証する？"

State the judgment clearly before proceeding.

### Step 2: Branch on verifier

If `VERIFIER=codex`, jump to [Step 2a — External verifier](#step-2a--external-verifier-codex).
Otherwise, continue with [Step 2b — In-process verifier](#step-2b--in-process-verifier).

### Step 2a — External verifier (Codex)

**Goal:** package the judgment as a context-stripped bundle, hand it to Codex, present Codex's verdict back to the user. Counter-argument enumeration, dismissal-condition declaration, and verification all happen *inside Codex*, not in your context.

**2a.1 Build the extraction bundle.** Populate this exact YAML schema using verbatim quotes only:

```yaml
fcot_extraction:
  claim: |
    <verbatim quote of the judgment under test — no paraphrasing>
  evidence_pointers:
    - <file:line | URL | verbatim user quote — one per line>
  domain_hint: |
    <one-line domain description, no agent framing>
```

Forbidden in any field (these re-introduce the bias the external verifier exists to escape):

- "reasoning so far", "considered alternatives", "given the context", "in my view", "I think"
- Synthesis prose summarizing what was discussed
- References to user identity, tone, or political framing
- Sibling hypotheses already rejected in this session

Escape hatches (these are different failure modes — distinguish them):

- **`claim` cannot be filled verbatim** → the judgment is too entangled with conversation context (deictic references like "this approach", or paraphrasing required to extract a standalone statement). Ask the user to restate the claim as a self-contained sentence, then re-invoke `/fcot`.
- **`evidence_pointers` is empty** (abstract architectural / value judgment with no concrete artifacts) → acceptable. Pass `evidence_pointers: []` and ensure `domain_hint` is rich enough for the verifier to reason from first principles. Tell the verifier explicitly that no evidence pointers were available, so it doesn't fabricate citations.

**2a.2 Write the verifier prompt to a file-pipe input.** Use the `/tmp/codex-io/` convention so the bundle is auditable across invocations. Tighten directory permissions because the bundle may contain proprietary or sensitive claims.

```bash
mkdir -p /tmp/codex-io
chmod 700 /tmp/codex-io   # readable only by you on shared machines
TS=$(date +%Y-%m-%d-%H%M%S)
# For non-ASCII claims (Japanese / CJK), use a romanized or English summary
# slug — the goal is filename greppability across past invocations, not
# semantic preservation of the original text.
SLUG="<short-kebab-slug-of-the-claim>"
INPUT=/tmp/codex-io/${TS}-fcot-${SLUG}.in.md
OUTPUT=/tmp/codex-io/${TS}-fcot-${SLUG}.out.md
```

Verifier prompt structure inside `INPUT` (markdown):

1. **The FCoT methodology brief**, tailored to the mode the user invoked:
   - **full mode** (default): instruct the verifier to produce ≥ 3 substantively different counter-arguments, each with a pre-declared dismissal condition, verification, and ✓/✗/? classification — followed by a Conclusion block.
   - **quick mode** (`/fcot quick` etc.): instruct the verifier to produce 1–3 prioritized counter-arguments with verification ≤ 2 sentences each and Conclusion ≤ 3 sentences. Pass the mode through; do not silently upgrade to full inside Codex.
   - In either case inline only the falsification logic itself (Step 2b — Enumerate / Verify / Conclude) and the Output Format block from this skill. **Do NOT inline Step 2 ("Branch on verifier") or Step 2a** — Codex *is* the external verifier, so verifier selection has already happened. Inlining the branching logic risks Codex re-triggering the external path inside its own process (infinite recursion or silent in-process degradation).
   - In the inlined Output Format spec, **omit the leading `Verifier:` placeholder line** — the parent agent prepends provenance after reading `OUTPUT` (see 2a.4). Codex's output should start with `## FCoT:`.
2. The extraction bundle from 2a.1.
3. The instruction: *use only your own investigation (web/file tools available in this process) — do not assume any context from a calling agent. No evidence pointers means none — do not fabricate.*

Do **NOT** include in `INPUT`: conversation history, your prior reasoning chain, sibling hypotheses, the user's stated preferences, or any synthesis you produced.

**2a.3 Invoke Codex with stdin redirect** (per the file-pipe protocol), and capture the exit code:

```bash
codex exec -s read-only < "$INPUT" > "$OUTPUT"
RC=$?
```

`-s read-only` keeps the verifier from mutating local state.

**2a.4 Read, validate, and present the output.** Treat the verifier run as failed if **any** of the following holds:

- `$RC -ne 0` (Codex exited non-zero — auth failure, model misconfig, network drop, sandbox denial)
- `[ ! -s "$OUTPUT" ]` (`OUTPUT` is empty)
- `OUTPUT` does not contain a `### Conclusion` section (malformed verdict)

On failure, do **NOT** fall back to in-process silently — report the failure (with exit code and a short excerpt from `OUTPUT` if any) and let the user decide whether to retry, switch verifier (`FCOT_VERIFIER=in-process /fcot ...`), or investigate the Codex installation. Silent fallback would defeat the bias-strip purpose.

On success, read `OUTPUT` and present it to the user. The parent agent prepends the provenance line so it appears exactly once (Codex's output is already configured to omit the `Verifier:` line per 2a.2):

```text
Verifier: codex (exit=0, file-pipe at <INPUT>)

<contents of OUTPUT>
```

### Step 2b — In-process verifier

The verifier is the same agent that produced the judgment. Acknowledge this bias risk to the user when starting, then proceed.

**2b.1 Enumerate Counter-Arguments.**

- **Minimum 3.** No upper limit.
- **Quality over quantity.** Each counter-argument must be substantively different. No padding — no rephrasing, no trivial variations of the same point.
- **Completeness is the bar** — have you missed any important counter-argument? Not "did you reach N?"
- For each counter-argument, **pre-define a dismissal condition** (FN bias): a specific, verifiable condition under which this counter-argument is eliminated.

**2b.2 Verify Dismissal Conditions.**

For each counter-argument, check whether its dismissal condition holds:

1. **First, reason logically.** If the condition can be settled by reasoning alone, do so.
2. **Investigate only when needed.** If the condition depends on facts (code, docs, data), then read the code, grep, check documentation — whatever it takes.
3. Classify each counter-argument: **✓ dismissed** or **✗ stands**.
4. **Inconclusive (?)** is permitted when the dismissal condition requires external facts. Before marking inconclusive, you MUST attempt up to 1-2 quick checks (a single grep, reading one file, checking one doc page). If those checks resolve it, classify as ✓ or ✗. If the question requires broader research beyond 1-2 quick checks, mark it **? inconclusive** and state the specific next action needed.
   Full classification: **✓ dismissed**, **✗ stands**, or **? inconclusive (requires: [specific action])**.

**2b.3 Conclude.**

Assess the overall result and act accordingly:

| Result | Action |
|--------|--------|
| All counter-arguments dismissed | **Judgment is sound.** State that the judgment survived falsification. Done. |
| Some counter-arguments stand (fixable) | **Judgment needs revision.** Present specific modifications or alternatives. Ask the user how to proceed. |
| Fundamental problems found | **Judgment should change.** Present the new judgment with reasoning. Ask the user to confirm. |
| Inconclusive counter-arguments remain | **Judgment is unresolved.** List what must be verified before the judgment can be confirmed or revised. Do not force a conclusion. |

## Output Format

Present analysis in this structure, regardless of which verifier produced it. The `Verifier:` line is always emitted exactly once by the agent that finalizes the output: the **parent agent** prepends it in external mode (after reading `OUTPUT` from Codex), and the **in-process verifier** emits it itself. When inlining this format into a Codex brief (per 2a.2), omit the leading `Verifier:` line — Codex's output starts with `## FCoT:` and the parent agent prepends the provenance.

```
Verifier: <codex | in-process> [provenance details]

## FCoT: [judgment summarized in one sentence]

### Counter-Arguments
| # | Counter-Argument | Dismissal Condition | Verification | Result |
|---|-----------------|---------------------|--------------|--------|
| 1 | ... | dismissed if ... | [reasoning or investigation result] | ✓ / ✗ / ? |
| 2 | ... | dismissed if ... | [reasoning or investigation result] | ✓ / ✗ / ? |
| 3 | ... | dismissed if ... | [reasoning or investigation result] | ✓ / ✗ / ? |

### Conclusion
**[Judgment is sound / Revision needed / Judgment changed / Unresolved — requires external verification]**

[Explanation]

[If revision/change: present alternatives + ask user]
```

**Quick mode output:** Same table structure. Each Verification cell ≤ 2 sentences. Conclusion ≤ 3 sentences. No detailed explanation section.

## Discipline

Do not shortcut this process. The value of FCoT is in the rigor, not the conclusion.

| Temptation | Reality |
|------------|---------|
| "My judgment is obviously right" | Then falsification will confirm it quickly. Do it anyway. |
| "These counter-arguments are weak" | Weak counter-arguments get dismissed fast. List them anyway. |
| "I already considered this" | Did you pre-declare dismissal conditions and verify them? If not, you didn't. |
| "This is taking too long" | Sycophantic agreement is faster. It's also wrong. |
| "3 counter-arguments is enough" | If you can think of a 4th substantive one, 3 is not enough. |
| "I'll mark it inconclusive to be safe" | Inconclusive is for genuinely unverifiable conditions. If a single grep or file read could resolve it, do that first. |
| "The user is just discussing FCoT, not invoking it" | Correct — FCoT only fires on explicit `/fcot` invocation. Discussion about the skill does not activate it. |
| "Codex is slow, let me skip to in-process" | The bias-strip is the point. Skipping it defeats the structural protection. Override only via explicit `FCOT_VERIFIER=in-process`, and surface that choice to the user. |
| "The external verifier disagreed with me — it must be wrong" | Disagreement between you and the external verifier is the *highest-value signal* this skill produces. Present both verdicts to the user; do not silently overrule. |
| "Codex failed, I'll quietly fall back to in-process" | Silent fallback hides the bias risk. Report the Codex failure and let the user decide. |
