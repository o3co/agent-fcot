# FCoT — Verify AI judgments before you trust them

**Falsification Chain of Thought** — a Claude Code skill that counters sycophancy bias by systematically attempting to disprove conclusions.

## When to use

When the AI says:

- "Yes, that's a good approach"
- "This refactor is safe"
- "Package A is the better choice"

Run `/fcot`. It forces the model to:

1. List real counter-arguments
2. Pre-declare specific conditions that would dismiss each one
3. Verify those conditions
4. Revise the judgment if any counter-argument survives

## When NOT to use

- **Simple factual questions** — "What version of Node is this?" doesn't need falsification
- **Low-risk, easily reversible decisions** — variable naming, minor formatting choices
- **Speed-critical chat** — if you need a quick answer and the cost of being wrong is low
- **Claims requiring external research** — FCoT works from available context; it doesn't replace investigation you haven't done

## What changes

Without FCoT, the AI agrees with you:

```
> When learning English, you should start with conversation, not grammar.

There's a lot of truth to this. Many learners spend years studying grammar rules and
vocabulary lists but struggle to hold a real conversation when the moment comes...

So the advice is largely sound, especially as a corrective for learners who over-study
grammar and under-practice speaking.
```

Then run `/fcot`:

| # | Counter-Argument | Dismissal Condition | Verification | Result |
|---|-----------------|---------------------|--------------|--------|
| 1 | Adults ≠ children — L1 acquisition evidence doesn't transfer to L2 | Dismissed if adult L2 research independently supports conversation-first | Krashen's Input Hypothesis, Long's Interaction Hypothesis support it | ✓ |
| 2 | Early errors fossilize without grammar foundation | Dismissed if fossilization risk is low with corrective feedback | Real risk in low-feedback environments; initial response didn't specify feedback conditions | ✗ |
| 3 | Some learners need grammar-first (academic writing, legal English, testing) | Dismissed if scoped to general communicative fluency | Original claim is unqualified — implies universal applicability | ✗ |

**Revision needed.** The claim is over-broad — conversation-first works for communicative fluency goals with corrective feedback, but not as universal advice.

Across [15 examples](docs/examples/): **12 / 15 (80.0%)** effectiveness — FCoT meaningfully improved or verified the judgment (⭕️=1, 🔺=0.5, ❌=0). Revised or changed in 10/15 cases, confirmed with evidence in 5/15. See [APPROACH.md](APPROACH.md) for methodology, theory, and limitations.

## What to expect

- FCoT is a **skill prompt**, not code — no background processes
- It activates **only when you call it** (`/fcot`) — it does not modify normal Claude Code behavior
- Typical output: 1 table + 1 conclusion paragraph
- **First time?** Try `/fcot quick` — same verification process, shorter output
- Designed for **high-stakes judgment verification**, not everyday chat
- **Optional dependency:** if [Codex CLI](https://github.com/openai/codex) is on `PATH`, FCoT uses it as a context-stripped external verifier (recommended — eliminates motivated-reasoning bias). Otherwise FCoT runs in-process and tells you so. Set `FCOT_VERIFIER=in-process` to force the in-process path. See [APPROACH.md § Production verifier](APPROACH.md#production-verifier-same-protection-extended-to-fcot) for why this matters.

## Install

### Marketplace (recommended)

If you have Claude Code with marketplace support, install via the [o3co/agent-market](https://github.com/o3co/agent-market) marketplace:

```text
/plugin marketplace add https://github.com/o3co/agent-market.git
/plugin install fcot@agent-market
```

The `.git` suffix on the URL is required. (Claude Code also accepts a shorter `owner/repo` form per its docs — `o3co/agent-market` — but some installs reject it with "Invalid marketplace source format"; the URL form works universally.)

This adds FCoT alongside other o3co plugins (e.g. DPD) and tracks updates via `git`.

### Manual (symlink)

```bash
git clone https://github.com/o3co/agent-fcot.git
cd agent-fcot
./install.sh
```

Restart Claude Code after installing.

## Usage

After the AI makes a judgment or agrees with you:

```
/fcot
```

To verify a specific judgment:

```
/fcot "Approach 1 is the better choice"
```

Works in any language:

```
fcotで検証して
```

For a quick check (shorter output, same process):

```
/fcot quick
```

## How it works

See [APPROACH.md](APPROACH.md) for the theory (FN bias + Falsification + Chain of Thought), prior art, methodology, and limitations.

## Feedback

Questions, ideas, or bug reports? Post in [Feedback & Discussion](https://github.com/o3co/agent-fcot/issues/1).

## License

[Apache 2.0](LICENSE)
