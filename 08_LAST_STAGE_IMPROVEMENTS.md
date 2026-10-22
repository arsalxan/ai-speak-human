# 08 — Last-Stage Improvements: What's Missing & How to Build Something Better

> **Turnitout is impressive, but it's not perfect.** This document identifies weaknesses, missing features, and opportunities to build a superior humaniser. These are your competitive advantages.

---

## Critical Weaknesses in the Current Approach

### 1. 🔴 No Grammar Verification Post-Transformation

**The Problem:** After running 16 stages of regex-based transformations, the output can contain grammatical errors:
- Voice transform might produce awkward passives: _"The results were analyzed by we"_
- Nominalization might produce broken sentences: _"An investigation of was conducted"_
- Clause reordering might mangle comma placement
- Sentence fusion might create run-on sentences

**The Improvement:**
- Add a **lightweight grammar checking pass** after Stage 16 (before output)
- Use `language-tool-python` (LanguageTool) — runs locally, no API needed
- Or use `gingerit` or `textblob` for basic grammar correction
- Focus on: subject-verb agreement, article usage, comma splices, run-on sentences
- **This single addition would dramatically improve output quality**

```python
# Post-pipeline grammar check (pseudocode)
import language_tool_python

tool = language_tool_python.LanguageTool('en-US')

def grammar_check(text):
    matches = tool.check(text)
    corrected = language_tool_python.utils.correct(text, matches)
    return corrected
```

---

### 2. 🔴 No Semantic Coherence Verification

**The Problem:** Synonym replacements can subtly shift meaning:
- _"The solution converges rapidly"_ → _"The answer amalgamates rapidly"_ (wrong context)
- _"The method is robust"_ → _"The method is stout"_ (wrong register)

**The Improvement:**
- Add a **context-aware synonym filter** using word embeddings (Word2Vec, GloVe, or spaCy)
- Before selecting a synonym, check if it has high cosine similarity with the original word **in the sentence context**
- Reject synonyms that would shift meaning too far

```python
# Context-aware synonym filtering (pseudocode)
import spacy
nlp = spacy.load("en_core_web_md")

def is_contextually_valid(original_word, synonym, sentence):
    doc = nlp(sentence)
    original_token = [t for t in doc if t.text == original_word]
    # Check if synonym fits the grammatical and semantic context
    synonym_doc = nlp(synonym)
    similarity = original_token[0].similarity(synonym_doc[0])
    return similarity > 0.5  # threshold
```

---

### 3. 🔴 No Paragraph-Level Coherence

**The Problem:** Turnitout processes text **line by line**. It has no awareness of:
- The argument flow across a paragraph
- Whether a transition phrase makes logical sense in context
- Whether fusing two sentences creates a semantic contradiction
- Whether the overall paragraph still reads naturally

**The Improvement:**
- Add a **paragraph-level preprocessing pass** that analyses the logical structure before transformation
- Track argument relationships (claim → evidence → conclusion)
- Select transitions that match the actual logical relationship between sentences
- Verify that sentence fusion only happens between semantically related sentences

---

### 4. 🟡 Regex-Based Voice Transform Is Fragile

**The Problem:** Voice transformation (active ↔ passive) using regex is error-prone with complex sentences:
- Doesn't handle irregular verbs well
- Struggles with compound subjects/objects
- Can't handle sentences with multiple clauses
- Doesn't account for tense consistency

**The Improvement:**
- Use **spaCy's dependency parser** for proper syntactic analysis
- Identify true subject-verb-object triples from the parse tree
- Handle irregular verb forms using a conjugation library (`lemminflect`, `pyinflect`)
- Only transform when the confidence in the parse is high

```python
# spaCy-based voice transform (pseudocode)
import spacy
nlp = spacy.load("en_core_web_sm")

def transform_voice(sentence):
    doc = nlp(sentence)
    # Find the root verb
    root = [t for t in doc if t.dep_ == "ROOT"][0]
    # Find subject and object
    subject = [t for t in root.children if t.dep_ in ("nsubj", "nsubjpass")]
    obj = [t for t in root.children if t.dep_ in ("dobj", "pobj")]
    # Properly restructure using dependency tree
    ...
```

---

### 5. 🟡 No Sentence-Level Quality Scoring

**The Problem:** There's no feedback loop. The engine doesn't know if a transformation improved or degraded the text. It blindly applies rules.

**The Improvement:**
- Add a **per-sentence quality scorer** that evaluates each transformation:
  - **Fluency score**: Does the output read naturally?
  - **Preservation score**: Is the meaning preserved?
  - **Perplexity delta**: Did perplexity actually increase?
- If a transformation degrades quality below a threshold, **roll it back**

```python
# Quality-gated transformation (pseudocode)
def safe_transform(text, transform_fn):
    original = text
    transformed = transform_fn(text)
    
    if fluency_score(transformed) < MIN_FLUENCY:
        return original  # Roll back
    if semantic_similarity(original, transformed) < MIN_SIMILARITY:
        return original  # Roll back
    
    return transformed
```

---

### 6. 🟡 No Adaptive Fire Rates

**The Problem:** Fire rates are static (e.g., 30% for voice transform). But different texts need different levels of transformation:
- A text with 80% AI score needs aggressive transformation
- A text with 20% AI score needs minimal changes
- Over-transforming already-human text makes it sound unnatural

**The Improvement:**
- Add an **adaptive fire rate system** that:
  1. Runs a quick perplexity/burstiness check on the input
  2. Calculates how much transformation is needed
  3. Dynamically adjusts fire rates per-stage
  4. Re-checks after each stage to avoid over-processing

---

### 7. 🟡 Limited Code Language Support

**The Problem:** The code comment humaniser supports ~12 languages but misses many:
- **Missing**: Ruby, Kotlin, Swift, PHP, Perl, R, Lua, Scala, Dart, Elixir, Haskell
- **Missing**: Jupyter notebooks (`.ipynb`) — JSON with embedded code+markdown
- **Missing**: Config files with comments (YAML, TOML, INI)

**The Improvement:**
- Add parsers for additional languages (most use `#`, `//`, or `/* */` — easy to add)
- Add Jupyter notebook support (parse JSON, extract markdown cells, process them)
- Add config file comment support

---

### 8. 🟢 No Built-in Perplexity Pre/Post Comparison

**The Problem:** Users have to manually copy-paste their text into external AI detectors to verify the result works.

**The Improvement:**
- Add a **built-in before/after comparison** that shows:
  - Input perplexity score (ZipPy or similar)
  - Output perplexity score
  - Burstiness comparison
  - N-gram overlap percentage with the original
  - Estimated AI detection probability

---

### 9. 🟢 No Cross-Sentence N-gram Awareness

**The Problem:** The n-gram audit (Stage 8) checks for surviving n-grams within individual lines, but doesn't catch n-grams that **span across line boundaries**:
```
Line 1: "...the proposed method demonstrates"
Line 2: "significant improvement over..."
Combined: "method demonstrates significant improvement over" → 5-gram SURVIVING
```

**The Improvement:**
- Add a **cross-sentence boundary n-gram audit** that checks for surviving n-grams at the junction of consecutive lines

---

### 10. 🟢 No PDF Support

**The Problem:** Many academic papers are distributed as PDFs. Currently, users must convert to `.docx` or `.tex` first.

**The Improvement:**
- Add PDF text extraction using `PyMuPDF` (`fitz`) or `pdfplumber`
- Process the extracted text
- **Note**: PDF → modified PDF is hard (preserving layout). More realistic: PDF → extract text → humanise → output as `.docx` or `.txt`

---

### 11. 🟢 No Multi-Language (Human Language) Support

**The Problem:** All rules are in English. Academic papers are written in many languages.

**The Improvement:**
- Create separate rule databases for other languages (Spanish, French, German, Chinese, Arabic, etc.)
- Add language detection (`langdetect` or `fasttext`)
- Route to appropriate language-specific rule set

---

### 12. 🟢 No Statistical Verification Against Source

**The Problem:** The engine doesn't compare its output against the original source text to calculate an estimated Turnitin similarity score.

**The Improvement:**
- Implement a **local Winnowing-style fingerprint comparison** that:
  1. Computes k-gram hashes of the original
  2. Computes k-gram hashes of the output
  3. Calculates Jaccard similarity
  4. Reports: "Estimated Turnitin similarity: ~X%"
- This would let users know if they need to re-run at higher intensity

---

## Priority Ranking for Building Your Own

| Priority | Improvement | Impact | Difficulty |
|:---------|:-----------|:-------|:-----------|
| 🔴 P0 | Grammar verification post-pipeline | Huge — prevents embarrassing errors | Easy (LanguageTool library) |
| 🔴 P0 | Context-aware synonym filtering | Huge — prevents wrong-meaning swaps | Medium (needs word embeddings) |
| 🟡 P1 | spaCy-based voice transform | Major — eliminates fragile regex | Medium (spaCy dependency parse) |
| 🟡 P1 | Sentence quality scoring + rollback | Major — prevents quality degradation | Medium |
| 🟡 P1 | Adaptive fire rates | Major — optimal transformation level | Medium |
| 🟡 P1 | Cross-sentence n-gram audit | Moderate — closes a gap | Easy |
| 🟢 P2 | Built-in before/after comparison | Nice UX — user confidence | Easy (ZipPy is ~50 lines) |
| 🟢 P2 | Local Winnowing similarity calculator | Nice UX — estimated Turnitin score | Medium |
| 🟢 P2 | PDF support | Expands format coverage | Medium |
| 🟢 P2 | Paragraph-level coherence | Better output quality | Hard |
| 🟢 P2 | More code languages | Expands coverage | Easy |
| 🟢 P3 | Multi-language support | Massive expansion | Hard (needs full rule sets per language) |

---

## Your Competitive Advantages

If you build these improvements into your humaniser, you'll have something **objectively better** than Turnitout:

1. **Grammar-verified output** — Turnitout doesn't have this
2. **Context-aware synonyms** — Turnitout uses blind dictionary lookup
3. **Quality-gated transformations** — Turnitout applies everything blindly
4. **Adaptive intensity** — Turnitout uses static fire rates
5. **Built-in verification** — Turnitout relies on external checkers
6. **Cross-line n-gram coverage** — Turnitout misses boundary n-grams
7. **PDF support** — Turnitout doesn't support PDF input

The core engine architecture (16-stage pipeline + JSON rules + placeholder masking) is sound. The improvements above are about making it **smarter and more reliable**.
