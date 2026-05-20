# SNOMED CT Markdown Tagger

A Google Colab notebook that performs **concept recognition and semantic classification** of clinical text written in Markdown, using the official SNOMED CT RF2 release as the reference terminology. Each word in an input document that corresponds to an active SNOMED CT concept is tagged with a semantic category and exported to a structured CSV for downstream analysis.

---

## Table of Contents

- [Overview](#overview)
- [Background and Motivation](#background-and-motivation)
- [Semantic Tag Taxonomy](#semantic-tag-taxonomy)
- [Architecture](#architecture)
- [Requirements](#requirements)
- [Usage](#usage)
- [Output Format](#output-format)
- [Design Decisions](#design-decisions)
- [Limitations and Known Constraints](#limitations-and-known-constraints)
- [Performance Characteristics](#performance-characteristics)
- [License Notice](#license-notice)

---

## Overview

`snomed_md_tagger.ipynb` is a self-contained Jupyter/Colab notebook that:

1. Accepts one or more Markdown (`.md`) files as clinical text input.
2. Accepts an official SNOMED CT RF2 release ZIP (supplied by the user under license).
3. Parses the SNOMED CT Snapshot files to extract active concepts, their English descriptions, and the IS-A hierarchy.
4. Constructs a high-throughput Aho-Corasick keyword matcher (via `FlashText`) from all active English Fully Specified Names and synonyms.
5. Tokenizes each Markdown document, splits it into pages at top-level headings, and identifies token spans that correspond to SNOMED CT concepts.
6. Classifies each matched concept into one of five semantic categories using a priority-ordered hierarchy traversal.
7. Exports one CSV per input file, then bundles all outputs into `tagged_outputs.zip`.

---

## Background and Motivation

SNOMED CT (Systematized Nomenclature of Medicine — Clinical Terms) is the world's most comprehensive multilingual clinical health terminology, maintained by SNOMED International. It organizes over 350,000 active concepts into a polyhierarchical directed acyclic graph using IS-A relationships, enabling precise and interoperable representation of clinical knowledge.

Named-entity recognition (NER) against controlled vocabularies — also called *dictionary-based* or *lexical* NER — is a foundational step in clinical natural language processing (NLP). Unlike statistical or neural NER models, dictionary-based matching is fully deterministic, requires no annotated training data, and produces concept identifiers that are directly actionable in standards-based workflows (HL7 FHIR, SNOMED expression constraints, CDS Hooks).

This notebook addresses a common research and documentation scenario: a clinician or biomedical author writes notes, summaries, case reports, or educational material in Markdown, and wishes to annotate that text with standardized SNOMED CT concept identifiers and semantic types — without standing up a full NLP server or commercial annotation service.

---

## Semantic Tag Taxonomy

The tagger classifies every matched concept into exactly one of five mutually exclusive categories. Where a concept belongs to more than one of the named hierarchies (which can occur in SNOMED CT's polyhierarchy), the following priority order governs:

| Priority | Tag | Anchoring SNOMED CT Concept | SCTID |
|----------|-----|-----------------------------|-------|
| 1 | `disease` | Disorder | `64572001` |
| 2 | `clinical finding` | Clinical finding (excluding Disorder) | `404684003` |
| 3 | `procedure` | Procedure | `71388002` |
| 4 | `drug` | Pharmaceutical / biologic product **or** Substance | `373873005`, `105590001` |
| 5 | `clinical term` | Any other active SNOMED CT concept | *(fallback)* |

The `disease` tag takes precedence over `clinical finding` because the Disorder hierarchy is a strict sub-hierarchy of Clinical finding in SNOMED CT, and the intent is to surface the most specific actionable category first.

---

## Architecture

The notebook is organized into nine numbered sections:

```
Section 1  Configuration         TAG_PRIORITY, concept TypeIds, working directories, MIN_TERM_LEN
Section 2  RF2 Loader            Streaming parsers for Concept, Description, and Relationship Snapshot files
Section 3  Hierarchy Classifier  BFS over IS-A edges to compute descendant sets; concept-to-tag map
Section 4  Matcher Builder       FlashText KeywordProcessor constructed from active English descriptions
Section 5  Markdown Processor    Page splitter, spaCy blank tokenizer, span-to-token assignment
Section 6  Sanity Check          In-memory mock SNOMED; asserts correct tagging and entity grouping
Section 7  Input Upload          Google Colab file-picker cells for the SNOMED ZIP and .md files
Section 8  Pipeline Execution    End-to-end run: unzip → load → build matcher → tag → write CSVs
Section 9  Output Preview        Head of the first CSV and tag-frequency distribution
```

### Data flow

```
SNOMED RF2 ZIP
      │
      ├─ sct2_Concept_Snapshot_*.txt         ─► active concept IDs
      ├─ sct2_Relationship_Snapshot_*.txt    ─► IS-A edge map ─► concept-to-tag map
      └─ sct2_Description_Snapshot-en*.txt  ─►┐
                                               ├─► FlashText KeywordProcessor
Markdown file(s)                               │
      │                                        │
      ├─ split on # / ## headings (pages)      │
      └─ spaCy blank tokenizer                ─►  tagged token rows ─► CSV
```

---

## Requirements

### Runtime

The notebook is designed for **Google Colab** (free tier is sufficient). It can also be run locally in any Jupyter environment.

### Python packages

```
flashtext
spacy
pandas
tqdm
```

These are installed automatically by the first code cell:

```python
!pip install -q flashtext spacy pandas tqdm
```

### SNOMED CT RF2 release

You must supply an official SNOMED CT RF2 **Snapshot** release ZIP yourself. The notebook will not download SNOMED CT on your behalf. Obtain the release from:

- [SNOMED International MLDS (Member Licensing & Distribution Service)](https://mlds.ihtsdotools.org/)
- Your country's National Release Centre

The ZIP must contain, somewhere in its directory tree, the three Snapshot files:

| Pattern | Purpose |
|---------|---------|
| `sct2_Concept_Snapshot_*.txt` | Active concept IDs |
| `sct2_Description_Snapshot-en*.txt` | English FSNs and synonyms |
| `sct2_Relationship_Snapshot_*.txt` | IS-A hierarchy edges |

When multiple files match a pattern (e.g., an Edition release that bundles both International and country-extension files), the largest file is selected automatically.

---

## Usage

### On Google Colab (recommended)

1. Open `snomed_md_tagger.ipynb` in Google Colab.
2. Select **Runtime → Run all**.
3. When prompted by Section 7a, upload your SNOMED CT RF2 release ZIP (e.g., `SnomedCT_InternationalRF2_PRODUCTION_20240101T120000Z.zip`).
4. When prompted by Section 7b, upload one or more `.md` files (hold Ctrl/Cmd to multi-select).
5. Wait for the pipeline to complete (~5–10 minutes on Colab free tier for a full International release).
6. `tagged_outputs.zip` downloads automatically to your browser.

### Locally (non-Colab)

Place files manually before running:

```
/content/
├── SnomedCT_*.zip          ← SNOMED RF2 release ZIP (one file)
└── markdown_inputs/
    ├── document1.md
    └── document2.md
```

Then execute all cells. The output bundle will be written to `/content/output/tagged_outputs.zip`.

---

## Output Format

Each input file `<stem>.md` produces one CSV named `<stem>.tagged.csv`. All CSVs are bundled into `tagged_outputs.zip`.

### CSV columns

| Column | Type | Description |
|--------|------|-------------|
| `number` | integer | 1-based word index within the file; resets per file, not per page |
| `word` | string | The surface form of the token as it appears in the source document |
| `tag` | string | Semantic category: one of `disease`, `clinical finding`, `procedure`, `drug`, `clinical term` |
| `page` | integer | Page number; pages are delimited by `#` or `##` headings; content before the first heading is page 1 |
| `entity_id` | integer | Groups tokens that belong to the same multi-word concept match (e.g., both tokens of *myocardial infarction* share the same `entity_id`) |
| `concept_id` | string | SNOMED CT concept identifier (SCTID) of the matched concept |

### Example

Given input containing `"Patient with acute myocardial infarction treated with aspirin."`:

| number | word | tag | page | entity_id | concept_id |
|--------|------|-----|------|-----------|------------|
| 4 | myocardial | disease | 1 | 1 | 22298006 |
| 5 | infarction | disease | 1 | 1 | 22298006 |
| 8 | aspirin | drug | 1 | 2 | 387458008 |

Only tokens that are part of a SNOMED CT match appear in the output. Unrecognized words produce no rows.

---

## Design Decisions

### FlashText over regex or spaCy NER

SNOMED CT's English description file contains approximately 1.4 million active terms. Constructing one regex pattern per term, or building a spaCy `PhraseMatcher` over all terms, does not scale to this volume without significant memory overhead. `FlashText` implements the Aho-Corasick string-matching algorithm, which performs multi-pattern matching in O(n) time relative to the length of the input text regardless of the number of patterns. This allows the full terminology to be loaded into the matcher while maintaining sub-second matching per document page.

### Streaming RF2 parsing

The SNOMED CT Description file can exceed 1.5 GB when uncompressed. All three RF2 files are parsed using Python's `csv.DictReader` with `quoting=csv.QUOTE_NONE`, which iterates line by line without materializing the entire file in memory. The matcher is built from a generator over this stream, keeping peak memory manageable on Colab's free tier (~12 GB RAM).

### Blank spaCy tokenizer

Tokenization uses `spacy.blank("en")`, which applies spaCy's English tokenization rules (whitespace and punctuation splitting) without loading any trained model weights. This keeps the environment lightweight and avoids a model download step. Punctuation and whitespace tokens are discarded before span-to-token assignment.

### Semantic tag from FSN stripped of trailing parenthetical

SNOMED CT Fully Specified Names carry a trailing semantic tag enclosed in parentheses (e.g., `Heart failure (disorder)`). The regex `\s+\([^()]+\)\s*$` strips this suffix before the term is added to the matcher, so that the matcher recognizes the phrase as it would appear in clinical prose rather than in the FSN form. Single-nested parentheticals only are stripped; this matches the RF2 convention for semantic tags.

### Priority-first tag assignment

When building the concept-to-tag map, each tag's descendant set is computed by BFS from its root concept(s) and written into the map only for concepts not already assigned a tag. Because `TAG_PRIORITY` is traversed in order (`disease` first), a concept that is a descendant of both Disorder and Clinical finding — which is structurally expected, since Disorder is a subtype of Clinical finding — will be assigned `disease`, not `clinical finding`. This is the intended behavior.

### Page numbering

Pages correspond to the first and second-level Markdown headings (`#`, `##`). Content preceding the first heading is treated as page 1. This convention aligns with common clinical document structures where top-level sections correspond to logical document divisions (e.g., History, Examination, Assessment).

### Minimum term length

Terms shorter than `MIN_TERM_LEN = 3` characters are excluded from the matcher. SNOMED CT contains short abbreviations and single-letter codes (e.g., `O`, `N`) that would produce unacceptably high false-positive rates in general prose matching.

---

## Limitations and Known Constraints

**No disambiguation.** The matcher returns the longest non-overlapping match. Where a phrase is ambiguous across multiple concepts (e.g., "aspirin" maps to both a substance and a product concept), only one concept ID is recorded — whichever was inserted into the `KeywordProcessor` last during construction. A production system would require disambiguation by context.

**Case-insensitive matching only.** The `KeywordProcessor` is configured with `case_sensitive=False`. This increases recall but may introduce false positives for short terms that are acronyms in one context and common words in another.

**English descriptions only.** The loader selects `sct2_Description_Snapshot-en*.txt`. Non-English SNOMED CT extensions or multilingual releases require modification of the file-selection pattern and potential encoding adjustments.

**Markdown structure assumptions.** The page splitter recognizes only `#` and `##` headings as page boundaries. Documents using `###` or deeper headings exclusively will be treated as a single page. HTML-embedded headings and ATX headings with no space after `#` are not recognized.

**No negation or context handling.** The tagger identifies concept mentions regardless of whether they are affirmed, negated, historical, or hypothetical ("no history of myocardial infarction" will still tag *myocardial infarction* as `disease`). Negation detection requires a dedicated NLP component (e.g., NegEx, ConText) not included here.

**Colab-specific file paths.** Section 7 and the working directory configuration assume `/content/` as the base path. On local Jupyter installations, these paths must be updated in Section 1.

**Runtime on large releases.** Processing a full SNOMED CT International RF2 release (approximately 350,000 active concepts and 1.4 million description terms) requires 5–10 minutes on Colab's free tier, primarily for ZIP extraction and matcher construction. This is a one-time cost per session; the matcher can be reused across multiple input documents within the same run.

---

## Performance Characteristics

| Step | Approximate time (Colab free tier, International RF2) |
|------|------------------------------------------------------|
| ZIP extraction | 2–4 minutes |
| Concept and relationship loading | ~1 minute |
| Matcher construction (1.4M terms) | 2–4 minutes |
| Per-document tagging | Seconds per file |

Peak memory usage is dominated by the `KeywordProcessor` object (typically 2–4 GB for the full International release). Colab's free tier provides ~12 GB RAM, which is sufficient.

---

## License Notice

SNOMED CT is a **proprietary, licensed terminology** owned by SNOMED International. Use of SNOMED CT content is subject to the SNOMED International Affiliate License Agreement. Membership in a SNOMED International Member country provides a royalty-free license for most uses; organizations in non-Member countries must obtain a fee-bearing license.

Obtain official SNOMED CT RF2 releases from the [Member Licensing & Distribution Service (MLDS)](https://mlds.ihtsdotools.org/) or your country's National Release Centre.

**This notebook does not distribute, embed, or download any SNOMED CT content.** You must supply the RF2 release file yourself. Use of this notebook does not substitute for compliance with your SNOMED CT license obligations.

---

## Citation

If you use this notebook in published research, please cite the SNOMED CT terminology and acknowledge SNOMED International in accordance with their attribution requirements.
