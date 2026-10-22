# 02 — Core Approach: Why Rule-Based Beats AI-Based

> **The fundamental insight**: AI detectors detect AI because AI-generated text has specific mathematical properties (low perplexity, low burstiness, high-probability tokens). Using an LLM to "humanise" text just creates *new* AI patterns — it's fighting fire with fire. A deterministic, rule-based engine doesn't introduce any AI patterns because it doesn't use AI.

---

## The Zero-AI Philosophy

### Why NOT use an LLM to paraphrase?

| Problem | What Happens |
|:--------|:-------------|
| **New AI fingerprint** | GPT rewriting GPT output still produces low-perplexity, statistically regular text → detectors catch it |
| **Hallucinations** | LLMs invent facts, alter numbers, change equations, introduce subtle errors |
| **Math/LaTeX destruction** | LLMs break custom LaTeX macros, rewrite equations incorrectly |
| **Code syntax corruption** | LLMs modify variable names, break indentation, introduce bugs |
| **Cost & latency** | API calls = money per token + network latency + rate limits |
| **Privacy leak** | Your research/IP gets sent to OpenAI/Anthropic/Google servers |
| **Cryptographic watermarks** | Claude (Anthropic), SynthID (Google) embed invisible statistical watermarks in their output — an LLM paraphraser preserves or creates new ones |

### Why DOES rule-based work?

| Advantage | Why |
|:----------|:----|
| **Zero AI fingerprint** | Deterministic regex/string operations produce zero statistical AI patterns |
| **Perfect accuracy** | Rules only do exactly what they're told — no hallucinations possible |
| **Math integrity** | LaTeX commands are masked before processing, restored after |
| **Code safety** | Only comments are modified; executable code is never touched |
| **Zero cost** | Python script processes thousands of lines in milliseconds — no API needed |
| **Total privacy** | Everything runs locally in-memory, nothing leaves the machine |
| **Speed** | ~0.001s per document vs ~5-30s for LLM paraphrasing |

---

## The Three-Pronged Attack Strategy

The humaniser targets all three detection layers simultaneously:

```
┌─────────────────────────────────────────────────┐
│              INPUT TEXT                          │
└───────────────────┬─────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ LEXICAL  │ │STRUCTURAL│ │ SEMANTIC │
│  LAYER   │ │  LAYER   │ │  LAYER   │
│          │ │          │ │          │
│ Synonyms │ │ Voice    │ │ Reorder  │
│ Phrases  │ │ Clause   │ │ Fuse     │
│ Determ.  │ │ Nominal  │ │ Split    │
│ N-gram   │ │ Apposit. │ │ Interp.  │
│ audit    │ │          │ │          │
└──────────┘ └──────────┘ └──────────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
┌─────────────────────────────────────────────────┐
│              OUTPUT TEXT                         │
│  • Different word hashes (Layer 1 defeated)     │
│  • Different syntax trees (Layer 2 defeated)    │
│  • Different semantic vectors (Layer 3 defeated)│
│  • High perplexity (AI detection defeated)      │
│  • High burstiness (AI detection defeated)      │
│  • High entropy (AI detection defeated)         │
└─────────────────────────────────────────────────┘
```

---

## Core Design Decisions

### 1. Deterministic, Seeded Randomness
- Every transformation uses a **seeded pseudo-random number generator** (`rng`)
- Same input + same seed → same output (reproducible)
- The seed provides variety but the system is NOT stochastic — it's predictable and testable

### 2. Probability-Gated Stages (Fire Rates)
```python
# Each stage has a "fire rate" — probability of activating per eligible line
if self.rng.random() > fire_rate:
    return text  # skip this stage for this line
```
- NOT every transformation fires on every sentence
- This creates **natural unevenness** in the output — which itself increases burstiness
- Default fire rates are calibrated per-stage (20–50%)

### 3. Logarithmic Intensity Mapping
```
User slider: 0% ──────────────── 75% (default) ──── 100%
                                  │
Internal aggressiveness:  0% ──── 20% ──────────── 100%
                                  ▲
                        Non-linear curve: (value/100)^0.75
```
- Linear slider is too crude — small changes at the low end should be subtle
- A **base-10 exponential volume curve** maps user-facing % to internal aggressiveness
- Default 75% → maps to ~20% internal aggressiveness (sweet spot)

### 4. Placeholder Masking (Protect-Then-Transform)
```
BEFORE:  "The solution to $\nabla^2 u = 0$ is well-known"
MASK:    "The solution to \x00PH0001\x00 is well-known"
TRANSFORM: "The answer to \x00PH0001\x00 is widely recognized"
RESTORE: "The answer to $\nabla^2 u = 0$ is widely recognized"
```
- All protected content (LaTeX math, `\cite{}`, code, URLs) is replaced with unique placeholders BEFORE any transformation
- Transformations operate on the masked text
- Placeholders are restored AFTER all transformations
- This guarantees protected content is NEVER modified

### 5. Asset Decoupling (Data ≠ Code)
```
rules/
├── synonyms.json          # 4,068+ word→synonym mappings
├── phrases.json           # Multi-word phrase rewrites
├── determiners.json       # Determiner swap rules
├── hedge_words.json       # Entropy-injecting hedge words
├── conjunctions.json      # Clause reorder triggers
├── protected_terms.json   # Never-modify terms
├── discourse_markers.json # Discourse rotation maps
└── academic/
    └── taxonomy/          # Citation discipline data
```
- ALL linguistic data lives in JSON files, NOT in Python code
- Contributors can expand synonyms without touching the engine
- Keeps the core script light, cohesive, and testable

### 6. Repetition Blocking
- **Synonym FIFO buffer** (size 15): tracks recently-used synonyms to prevent repetition
- **Determiner recently-used queue**: prevents cycling the same formal determiner
- **Discourse marker tracking**: never uses the same replacement twice in a document
- **Appositive single-use enforcement**: each appositive injected at most once per document
- **Sentence context check**: synonym candidates already present in the sentence are filtered out

---

## Key Mathematical Insight: Why Changing 1 Word in 5 Destroys the Hash

```
k-gram (k=5): "we analyze the heat equation"  → Hash A
Changed:       "we examine the heat equation"  → Hash B (COMPLETELY different)

Because the Karp-Rabin rolling hash is:
h(t) = (c₁ × b^(k-1) + c₂ × b^(k-2) + ... + cₖ × b^0) mod p

One different character in ANY position cascades through the entire polynomial,
producing a completely different hash value.
```

This means:
- Changing 1 word out of every 5 → **every single k-gram in the document gets a new hash**
- Turnitin's fingerprint database has zero matches
- Similarity score → 0%

The humaniser doesn't need to change every word — it needs to ensure that **no 5+ consecutive words survive unchanged**.
