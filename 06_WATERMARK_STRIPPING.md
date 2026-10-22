# 06 — Watermark & Tracker Stripping

> **Beyond similarity and AI detection, modern AI providers embed invisible tracking in their output.** This document explains what these watermarks are and how a rule-based engine can strip them — something LLM paraphrasers fundamentally cannot do.

---

## The 2026 Tracking Landscape

As of 2026, AI providers embed multiple layers of tracking mandated by the EU AI Act (Article 50):

| Provider | Watermark Technology | How It Works |
|:---------|:--------------------|:-------------|
| **Google (Gemini)** | SynthID-Text | Cryptographic token-selection bias during generation |
| **Anthropic (Claude)** | Statistical watermarking | Invisible statistical patterns in token distributions |
| **OpenAI (ChatGPT)** | Undisclosed | Suspected token probability shaping |
| **All providers** | Zero-Width Unicode | Invisible characters (`U+200B`, `U+200C`, `U+200D`, `U+2060`) injected between words |
| **All providers** | C2PA Metadata | Content Credentials embedded in document/image metadata |

---

## 1. Cryptographic Token Watermarks (SynthID, Claude)

### How They Work

When an LLM generates text, at each token position it calculates probabilities for all possible next tokens. Normally it samples from these probabilities. Watermarking **biases the sampling**:

```
Without watermark:
  Token probabilities: {"the": 0.15, "a": 0.12, "this": 0.10, ...}
  Selected: sampled normally

With SynthID watermark:
  Token probabilities are partitioned into "green" and "red" lists using a secret key
  The model preferentially selects tokens from the "green" list
  Over many tokens, the statistical bias becomes detectable with the secret key
```

**Key insight**: The watermark is a statistical property distributed across the ENTIRE text. You can't see it. You can't find it by looking at individual words. It exists only in the aggregate statistical distribution of token choices.

### How Rule-Based Transformation Destroys It

```
Original (watermarked):   "The method demonstrates significant improvement"
                           ↑ green    ↑ green      ↑ green      ↑ green

After synonym replacement: "The technique elucidates appreciable advancement"
                            ↑ ???      ↑ ???        ↑ ???         ↑ ???
```

- Each replaced word is a NEW token that was NOT selected by the watermarked LLM
- The replacement was selected by a deterministic rule engine, NOT by an LLM
- The statistical bias pattern (green/red token distribution) is **shattered**
- The watermark detector's secret key no longer produces a match

**Why LLM paraphrasers FAIL here**: An LLM paraphraser generates new tokens using its OWN watermarked sampling process. It may remove the original watermark but introduces a NEW watermark from its own provider.

**Why rule-based SUCCEEDS**: Rule-based synonym selection has ZERO correlation with any LLM's token probability distribution. It doesn't use token probabilities at all — it picks from a curated JSON dictionary.

---

## 2. Zero-Width Unicode Character Purge

### What Are Zero-Width Characters?

Invisible Unicode characters that take up zero visual space but are present in the text data:

| Character | Code Point | Name | Purpose in Tracking |
|:----------|:-----------|:-----|:-------------------|
| ​ | `U+200B` | Zero-Width Space | Inserted between words as invisible markers |
| ‌ | `U+200C` | Zero-Width Non-Joiner | Used in specific positions to encode bits |
| ‍ | `U+200D` | Zero-Width Joiner | Creates unique patterns per document |
| ⁠ | `U+2060` | Word Joiner | Additional tracking bit |
| ﻿ | `U+FEFF` | BOM / Zero-Width No-Break Space | Document-level identifier |

### How They're Used for Tracking

```
Visible text: "The method produces accurate results"
Actual bytes: "The\u200B method\u200C produces\u200B accurate\u200D results"
                    ^^^        ^^^^          ^^^          ^^^^
              Invisible tracking characters encoding a unique ID
```

Each AI service can inject a unique pattern of these characters that identifies:
- Which model generated the text
- Which user account requested it
- A timestamp or session ID

### How to Strip Them

```python
import re

# Comprehensive zero-width character removal
ZERO_WIDTH_PATTERN = re.compile(
    '[\u200b\u200c\u200d\u2060\ufeff'  # Common zero-width chars
    '\u00ad'                             # Soft hyphen
    '\u200e\u200f'                       # LTR/RTL marks
    '\u202a-\u202e'                      # Directional formatting
    '\u2066-\u2069'                      # Directional isolates
    ']'
)

def strip_zero_width(text: str) -> str:
    return ZERO_WIDTH_PATTERN.sub('', text)
```

**This is a simple regex operation** — no AI needed. Just remove every character in the zero-width Unicode range.

### Why They Also Break Code

Zero-width characters injected into source code can:
- Cause `SyntaxError` in Python (invisible character in variable name)
- Break string comparisons (strings look identical but contain hidden chars)
- Corrupt hashes and checksums
- Trigger unexpected behavior in parsers

---

## 3. C2PA Metadata Stripping

### What Is C2PA?

Content Credentials (C2PA — Coalition for Content Provenance and Authenticity) is a metadata standard that embeds provenance information inside files:

```
document.docx
├── [Content]
│   └── ... (your text, images, etc.)
└── [C2PA Manifest]
    ├── claim.generator: "Claude 3.5 Sonnet"
    ├── claim.created: "2026-10-22T06:00:00Z"
    ├── claim.signature: <cryptographic signature>
    └── actions:
        └── c2pa.created: {softwareAgent: "Anthropic API"}
```

### How to Strip It

For `.docx` files (which are ZIP archives):
```python
import zipfile
import io

def strip_c2pa_metadata(docx_bytes: bytes) -> bytes:
    """Remove C2PA manifests from a .docx file."""
    input_zip = zipfile.ZipFile(io.BytesIO(docx_bytes))
    output_buffer = io.BytesIO()
    
    with zipfile.ZipFile(output_buffer, 'w') as output_zip:
        for item in input_zip.namelist():
            # Skip C2PA manifest files
            if 'c2pa' in item.lower() or 'content_credentials' in item.lower():
                continue
            output_zip.writestr(item, input_zip.read(item))
    
    return output_buffer.getvalue()
```

For images and other media, similar metadata stripping applies to EXIF, XMP, and IPTC metadata fields.

---

## 4. Em-Dash & En-Dash Stripping (LLM Signature Indicators)

### The Problem

LLMs (especially Claude and GPT-4) have a strong tendency to use em-dashes (`—`) and en-dashes (`–`) instead of standard hyphens or commas. This is a known stylometric indicator:

```
AI-typical:  "The method—which was first proposed—has shown significant results"
Human-typical: "The method, which was first proposed, has shown significant results"
```

### The Fix

```python
def strip_llm_dashes(text: str) -> str:
    # Replace em-dashes with commas or remove them
    text = text.replace('—', ', ')
    text = text.replace('–', '-')
    return text
```

Simple string replacement — another indicator that AI detectors look for, trivially removed without AI.

---

## Summary: Why Rule-Based Engines Win at Watermark Removal

| Watermark Type | LLM Paraphraser | Rule-Based Engine |
|:---------------|:----------------|:------------------|
| **SynthID/Claude cryptographic** | ❌ Creates NEW watermark from its own provider | ✅ Shatters statistical pattern (no LLM involvement) |
| **Zero-width Unicode** | ❌ Often preserves invisible chars | ✅ Simple regex removal |
| **C2PA metadata** | ❌ May add its own metadata | ✅ ZIP-level manifest removal |
| **Em-dash signatures** | ❌ May introduce its own stylistic markers | ✅ Simple string replacement |
