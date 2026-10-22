# 07 — System Architecture & Project Structure

> **How to structure your own humaniser codebase.** This document covers the module breakdown, data flow, configuration system, and recommended project layout.

---

## Recommended Project Structure

```
humaniser/
├── src/
│   └── humaniser/
│       ├── __init__.py
│       ├── core/
│       │   ├── __init__.py          # MUST be empty (docstring only) — avoid circular imports
│       │   ├── modifier.py          # THE core engine — 16-stage transformation pipeline
│       │   ├── rules.py             # JSON rule loader (dynamic, recursive)
│       │   └── config.py            # Configuration management (JSON + env vars)
│       │
│       ├── parsers/
│       │   ├── __init__.py
│       │   ├── doc_processor.py     # .docx XML run-level parser
│       │   ├── latex_parser.py      # .tex zone classifier & placeholder masker
│       │   ├── code_tokenizer.py    # AST-style comment isolator for source code
│       │   ├── markdown_parser.py   # Markdown fence-aware processor
│       │   └── zip_handler.py       # In-memory ZIP archive processor
│       │
│       ├── citation/
│       │   ├── __init__.py
│       │   ├── citation_engine.py   # Citation generation engine (decoupled from UI)
│       │   └── bibtex_generator.py  # BibTeX / JSON / LaTeX reference output
│       │
│       ├── watermark/
│       │   ├── __init__.py
│       │   ├── zero_width.py        # Zero-width Unicode character stripper
│       │   ├── c2pa_stripper.py     # C2PA metadata removal
│       │   └── dash_normalizer.py   # Em-dash / en-dash LLM indicator removal
│       │
│       ├── detector/
│       │   ├── __init__.py
│       │   └── zippy_checker.py     # ZipPy compression perplexity checker
│       │
│       └── ui/
│           ├── __init__.py
│           ├── common.py            # Shared UI components (header, footer, traffic guard)
│           ├── humanizer_ui.py      # Humaniser page
│           ├── citation_ui.py       # Citation inserter page
│           ├── checker_ui.py        # AI pattern checker page
│           └── feedback_ui.py       # Bug reporter page
│
├── rules/
│   ├── synonyms.json
│   ├── phrases.json
│   ├── determiners.json
│   ├── hedge_words.json
│   ├── conjunctions.json
│   ├── protected_terms.json
│   ├── discourse_markers.json
│   ├── qualifiers.json
│   ├── voice_verbs.json
│   ├── nominalization.json
│   ├── appositives.json
│   ├── transition_phrases.json
│   └── academic/
│       └── taxonomy/
│           └── ... (discipline subfolders)
│
├── configs/
│   └── global_config.json           # Single source of truth for all settings
│
├── tests/
│   ├── core/
│   │   ├── test_modifier.py
│   │   └── test_rules.py
│   └── parsers/
│       ├── test_doc_processor.py
│       └── test_latex_parser.py
│
├── streamlit_app.py                  # Web app entry point
├── requirements.txt
└── .env                              # Environment variable overrides
```

---

## Data Flow Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         USER INPUT                                 │
│     (text / .docx / .tex / .py / .zip)                           │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                    FORMAT DETECTION                                │
│     Identify file type → route to appropriate parser              │
└───────────────────────────┬────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ DOCX     │ │ LaTeX    │ │ Code     │
        │ Parser   │ │ Parser   │ │ Tokenizer│
        │          │ │          │ │          │
        │ Extract  │ │ Classify │ │ Isolate  │
        │ run text │ │ zones    │ │ comments │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │             │             │
             └─────────────┼─────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│                  WATERMARK STRIPPING                               │
│     • Zero-width Unicode purge                                    │
│     • Em-dash / en-dash normalisation                             │
│     • C2PA metadata removal (for .docx)                           │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│               TRANSFORMATION PIPELINE (modifier.py)               │
│                                                                    │
│  ┌─ Stage 1:  Protect/mask LaTeX, math, code                     │
│  ├─ Stage 2:  Phrase rewrites (multi-word)                        │
│  ├─ Stage 3:  Synonym substitution (single-word)                  │
│  ├─ Stage 4:  Clause reordering                                   │
│  ├─ Stage 5:  Determiner swaps                                    │
│  ├─ Stage 6:  Sentence splitting                                  │
│  ├─ Stage 7:  Hedge word injection                                │
│  ├─ Stage 8:  N-gram chain audit (safety net)                     │
│  ├─ Stage 9:  Voice transform (active ↔ passive)                  │
│  ├─ Stage 10: Sentence fusion                                     │
│  ├─ Stage 11: Transition injection                                │
│  ├─ Stage 12: Clause word reordering                              │
│  ├─ Stage 13: Nominalization                                      │
│  ├─ Stage 14: Appositive injection                                │
│  ├─ Stage 15: Discourse marker rotation                           │
│  └─ Stage 16: Restore/unmask LaTeX, math, code                   │
│                                                                    │
│  (Optional) Stage 17: Citation insertion                           │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                 FORMAT WRITER                                      │
│     • DOCX: write back to XML runs (preserving formatting)        │
│     • LaTeX: write back to .tex (zones restored)                  │
│     • Code: write back comments (code untouched)                  │
│     • ZIP: re-package into archive                                │
└───────────────────────────┬────────────────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────────────────┐
│                  OUTPUT PACKAGING                                  │
│     • Single file download                                        │
│     • Branded ZIP bundle with timestamp                           │
│     • Statistics report (changes count, per-stage breakdown)      │
└────────────────────────────────────────────────────────────────────┘
```

---

## Configuration System

### Single Source of Truth: `configs/global_config.json`

```json
{
  "transformation": {
    "synonym_aggressiveness": 0.20,
    "enable_voice_transform": true,
    "voice_transform_rate": 0.30,
    "enable_sentence_fusion": true,
    "sentence_fusion_rate": 0.25,
    "enable_transition_inject": true,
    "transition_inject_rate": 0.25,
    "enable_word_reorder": true,
    "word_reorder_rate": 0.20,
    "enable_nominalization": true,
    "nominalization_rate": 0.20,
    "enable_appositive": true,
    "appositive_rate": 0.35,
    "enable_discourse_rotate": true,
    "discourse_rotate_rate": 0.50
  },
  "ui_flags": {
    "show_ai_detection_in_remover": true,
    "show_ai_checker_tab": true
  },
  "whitelisted_users": []
}
```

### Override Hierarchy
```
Environment Variable  →  overrides  →  JSON config  →  overrides  →  Code defaults
(highest priority)                                                    (lowest priority)
```

### Environment Variables
```bash
# Per-stage toggles
TURNITOUT_VOICE_TRANSFORM=true
TURNITOUT_SENTENCE_FUSION=true
TURNITOUT_TRANSITION_INJECT=true
TURNITOUT_WORD_REORDER=true
TURNITOUT_NOMINALIZATION=true
TURNITOUT_APPOSITIVE=true
TURNITOUT_DISCOURSE_ROTATE=true

# Per-stage fire rates (0.0 to 1.0)
TURNITOUT_VOICE_RATE=0.30
TURNITOUT_FUSION_RATE=0.25
TURNITOUT_TRANSITION_RATE=0.25
TURNITOUT_REORDER_RATE=0.20
TURNITOUT_NOMINAL_RATE=0.20
TURNITOUT_APPOSITIVE_RATE=0.35
TURNITOUT_DISCOURSE_RATE=0.50
```

---

## The Rule Loader (`rules.py`)

```python
# Pseudocode for the dynamic rule loader
import json
import os

def load_json_rule(filepath):
    """Load a JSON rule file, stripping __prompt__ metadata."""
    with open(filepath) as f:
        data = json.load(f)
    
    if isinstance(data, dict):
        data.pop("__prompt__", None)
    elif isinstance(data, list):
        if data and isinstance(data[0], dict) and "__prompt__" in data[0]:
            data = data[1:]
    
    return data

def load_academic_taxonomy(taxonomy_dir):
    """Recursively load all taxonomy JSON files."""
    taxonomy = {}
    for root, dirs, files in os.walk(taxonomy_dir):
        for f in files:
            if f.endswith('.json'):
                path = os.path.join(root, f)
                taxonomy[f.replace('.json', '')] = load_json_rule(path)
    return taxonomy

# Load all rules at module level
SYNONYMS = load_json_rule("rules/synonyms.json")
PHRASES = load_json_rule("rules/phrases.json")
DETERMINERS = load_json_rule("rules/determiners.json")
HEDGE_WORDS = load_json_rule("rules/hedge_words.json")
CONJUNCTIONS = load_json_rule("rules/conjunctions.json")
PROTECTED_TERMS = load_json_rule("rules/protected_terms.json")
ACADEMIC_TAXONOMY = load_academic_taxonomy("rules/academic/taxonomy/")
```

---

## The Modifier Class (`modifier.py`)

```python
# Pseudocode for the core engine
import re
import random

class TextModifier:
    def __init__(self, config, rules, seed=None):
        self.config = config
        self.rules = rules
        self.rng = random.Random(seed)
        
        # Counters for statistics
        self.synonym_count = 0
        self.phrase_rewrite_count = 0
        self.voice_transform_count = 0
        # ... etc.
        
        # Repetition blockers
        self._synonym_fifo = []       # size 15
        self._used_appositives = set()
        self._used_discourse_replacements = {}
    
    def modify_line(self, text: str) -> str:
        """Run the full 16-stage pipeline on a single line of text."""
        text = self._protect_latex(text)          # Stage 1
        text = self._apply_phrase_rewrites(text)  # Stage 2
        text = self._apply_synonyms(text)         # Stage 3
        text = self._reorder_clauses(text)        # Stage 4
        text = self._swap_determiners(text)       # Stage 5
        text = self._split_compound(text)         # Stage 6
        text = self._insert_hedge_words(text)     # Stage 7
        text = self._break_ngram_chains(text)     # Stage 8
        text = self._transform_voice(text)        # Stage 9
        text = self._fuse_sentences(text)         # Stage 10
        text = self._inject_transitions(text)     # Stage 11
        text = self._reorder_within_clause(text)  # Stage 12
        text = self._nominalize(text)             # Stage 13
        text = self._inject_appositives(text)     # Stage 14
        text = self._rotate_discourse(text)       # Stage 15
        text = self._restore_latex(text)          # Stage 16
        return text
```

---

## Critical Architectural Invariants

### 1. `core/__init__.py` Must Be Empty
```python
# core/__init__.py
"""Core transformation engine modules."""
# NEVER import anything here — causes circular import loops with rules.py and config.py
```

### 2. Lazy Model Loading
- NEVER import heavy libraries (spaCy, WordNet, NLP models) at the module level
- Load them lazily, only when actually needed
- Heavy models consume ~400MB RAM and 2-3 seconds on cold start

### 3. Stateless Processing
- No files written to disk during processing
- Everything in-memory (BytesIO for ZIP packaging)
- No document logging, no storage, no data retention

### 4. Pipeline Callback Decoupling
- Document parsers accept a `pipeline_callback` function
- The callback is `modifier.modify_line`
- Parsers never import UI, CLI, or web framework dependencies
