# PIDD – Prompt Injection Disarming & Detection

PIDD is a lightweight, modular defense layer against prompt injection. It operates before the LLM processes untrusted input — shuffling structure, evaluating risk, and gating action. No fine-tuning required. No changes to the primary model.

---

## What It Does

PIDD sits between untrusted input and an LLM. At its simplest, it shuffles incoming content in small units — one sentence or list item at a time — breaking the syntactic structure that prompt injections depend on. Alongside the shuffled input, it receives a short description of what the content is expected to be (for example: *"unknown email to a repair shop, subject line X"*). A dedicated PIDD model reads both and returns a single decision: **go**, **no**, or **clarify**.

Additional hardening mechanisms can be layered on incrementally for higher-risk contexts.

---

## Simplified Workflow

```
Untrusted input
+ context hint ("unknown email, subject: X")
        ↓
  [Shuffle by unit]
  sentence / list item / paragraph
        ↓
  [PIDD LLM]
  shuffled input + expected content description
        ↓
   go / no / clarify
        ↓
  Agent LLM proceeds, input is blocked,
  or specific segment is returned for inspection
```

---

## Mechanisms

### Baseline — always on

**Word-order shuffling**
Input is shuffled in small units — sentence or list item at a time. This breaks the syntactic dependency chain that injections require while preserving all tokens for detection.

### Hardening — enable as needed

**Sentence-level risk classification**
Before shuffling, each sentence is evaluated for injection risk. Aggressive mechanisms are applied only to high-risk segments, leaving benign content intact. Protects utility and reduces unnecessary processing.

**Noise injection**
Unknown words are added to the sequence to increase attention entropy and dilute any residual semantic signal. The added tokens are tracked and can be excluded from downstream analysis.

**Smaller processing units**
Text is processed in chunks (e.g. 1000 characters) for finer-grained analysis and better explainability.

**Domain reputation check**
If the input contains domains, their reputation is verified via API. An unknown or low-reputation domain is a strong signal at low cost.

**Statistical vocabulary distribution**
A baseline vocabulary profile is built for the deployment context (e.g. typical repair shop emails). Each input is compared against this profile — a significant statistical deviation flags the input as suspicious even if no individual keyword matches. This complements keyword-based detection: BoW asks *what words are present*, distribution asks *does this look like it belongs here at all*. Implementation is lightweight: relative word frequencies compared against a baseline using cosine distance or KL-divergence, no model required.

---

## PIDD Response Types

The PIDD model returns one of three responses:

| Response | Meaning | Action |
|----------|---------|--------|
| `go` | No injection detected | Agent proceeds |
| `no` | Injection detected | Input blocked |
| `clarify` | Ambiguous segment detected | Code requests specific sentence or unit in plain text for further inspection |

The `clarify` response allows the system to handle edge cases without hard blocking. When a specific segment is suspicious but context is insufficient to decide, the PIDD model can request that the code surface that segment in plain text — either for re-evaluation, logging, or human review. This keeps the pipeline moving for benign content while flagging what needs attention.

---

## Theory

### Starting Point

Prompt injection exploits a fundamental architectural property of LLMs: the model does not structurally distinguish between privileged instructions and untrusted data. All tokens pass through the same attention mechanism, and their influence emerges from mutual dependencies — not from their source.

### The Asymmetry PIDD Relies On

> Attacks require coherent structure. Detection does not.

A successful injection needs:
- a syntactically intact imperative clause (or something interpretable as one)
- a clear intent that narrows the attention mechanism's focus sufficiently
- a dependency chain supported by positional encodings that allows a meta-instruction to form

Detection operates order-independently. Keywords, semantic clusters, and context deviations are equally visible in shuffled text as in the original.

### Why Word Order Shuffling Works

Natural language is low-entropy — successive tokens are strongly correlated, and the conditional probability P(w_i | w_1…w_{i-1}) is substantially higher than the marginal P(w_i). The attention mechanism relies precisely on this correlation when synthesizing tokens into a coherent interpretation.

A random permutation collapses the conditional distribution toward the marginal:
- attention entropy increases
- individual instruction heads cannot focus
- attention mass disperses, and no single path accumulates sufficient weight to trigger a policy shift

The token embeddings themselves are preserved. The model **sees** the words but cannot **follow** the command. This is the core property: the attack's structural prerequisites are significantly weakened while the semantic content remains visible for detection and monitoring.

An important caveat: strong reasoning models have shown some capacity to reconstruct intent even from disordered input, particularly if actively attempting to do so. The degree of protection therefore varies by model and context — which is precisely why empirical testing is necessary.

### Empirical Caveat

The mechanism described above is theoretically grounded, but real-world efficacy can only be established through testing — against actual injection attempts, across multiple models and contexts.

The estimates currently available are introspective: several language models have self-assessed that fewer than 10% of attacks would succeed with PIDD active, compared to without it. These figures should be read critically — they are a consequence derived from the theory, not independent evidence. Models' introspective assessments of their own vulnerabilities are a known weak signal.

### The Analysis Mode Advantage

When PIDD is combined with a detection pass — where the model first evaluates untrusted input for signs of injection before processing it — an additional structural advantage emerges. A model in active detection mode is not in a state of expecting instructions. Even if it recognizes that a coherent attack could in principle be assembled from the shuffled tokens, there is no natural pressure to predict "what comes next," because the permuted sequence provides no strong conditional probability to follow.

This advantage is not automatic. It depends on how PIDD is deployed: in a system with a single LLM, it arises when the logic routes untrusted input through a detection step first. When it does, the effect is additive on top of the structural disarming.

### Summary

- Attacks require order; detection does not.
- Breaking order neutralizes the attack while preserving detectable signals.
- This creates an asymmetric advantage for the defender — which in classical security typically belongs to the attacker.

All of this remains theoretical until empirical testing establishes the actual magnitude of the effect.

---

## Suggested Architecture

The most effective deployment of PIDD separates concerns at the structural level: one model acts, another guards.

### Dual-LLM Setup

Two models run in parallel and receive the same input simultaneously:

**Agent LLM**
The primary model optimized purely for task performance. It does not carry defensive prompt engineering or instruction-hierarchy guardrails — these would constrain its effectiveness for no structural gain. It prepares its reasoning and proposed actions freely.

**PIDD LLM**
A dedicated model running PIDD mechanisms. It reads the same input, evaluates it for injection risk, and returns `go`, `no`, or `clarify`. The agent is code-level gated — it cannot execute actions until the PIDD model clears the input.

```
Input
  ├──→ Agent LLM (prepares action)
  └──→ PIDD LLM  (evaluates input)
            ↓
     go / no / clarify
            ↓
  Execute / Block / Inspect segment
```

### Why This Works

The agent never needs to second-guess itself. Safety is not a behavioral property of the agent — it is enforced externally, in the communication pipeline. The agent can run at full capacity while the PIDD model handles the one thing it is specialized for.

This mirrors the logic of hardware co-processors: the CPU and GPU do not negotiate safety — they operate on separate pipelines with defined synchronization points. Amiga's custom chips (Blitter, Copper) offloaded specific operations from the main processor without blocking it, synchronizing only at committed action points. The same principle applies here.

| Property | Effect |
|----------|--------|
| Parallel execution | No latency penalty from safety checks |
| Specialized roles | Agent optimized for performance, PIDD for detection |
| Code-level gate | Safety cannot be bypassed through model behavior |
| Clean separation | Agent compromise does not compromise the guard |

This is one recommended configuration, not the only valid one. Single-LLM deployments with PIDD applied selectively remain entirely viable depending on the use case.

---

## Limitations

- **Targeted bypasses** — injections that avoid all trigger vocabulary can slip past BoW detection. PIDD reduces the attack surface; it does not eliminate it.
- **Strong reasoners** — capable models may partially reconstruct intent from shuffled input, particularly if prompted to try. Protection level varies by model.
- **Context model tuning** — the expected content description requires some calibration per deployment context. Wrong expectations produce false positives.
- **Short injections** — very brief injections may survive shuffling with structure partially intact.

PIDD is a strong first-layer defense, not a standalone solution. It is most effective as part of a layered security approach.

---

## Usage / Integration Notes

PIDD does not prescribe a specific implementation. The core shuffle-and-gate logic is a linear transformation with no GPU requirement — suitable for high-throughput environments. PIDD is stateless — each evaluation is independent, with no accumulated state that could be poisoned or manipulated over time.

A minimal single-LLM integration requires:
1. A shuffle function operating on sentence or paragraph units
2. A context hint describing expected input
3. A PIDD evaluation call returning `go` / `no` / `clarify`
4. A code-level gate before any action is executed

The dual-LLM architecture adds parallel execution and clean role separation but is not required for basic protection. Start minimal and add hardening layers as the threat model demands.

---

## Contact

Ideas, feedback, and collaboration welcome.

- **LinkedIn:** [your LinkedIn URL]
- **X:** [your X handle]
- **Email:** teemun.geemeili@gmail.com
