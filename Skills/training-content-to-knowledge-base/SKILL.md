# Training Content to Knowledge Base

## Purpose

Converts training tip sheets, job aids, and procedural documents (typically PDFs with annotated screenshots) into structured, LLM-readable knowledge base content with an accompanying eval question set for quality assurance.

## When to Use

Trigger this skill when the user:
- Asks to convert a training document, tip sheet, or job aid into something "LLM-readable", "AI-friendly", or suitable for a "knowledge base"
- Wants to make a document with embedded images/screenshots understandable to an LLM
- Asks to generate eval questions to test an LLM's ability to reason over training content
- Mentions building a knowledge base from training materials
- References "tip sheet", "job aid", "quick reference guide", "SOP", or "how-to guide" in the context of LLM consumption

## Inputs

- One or more training documents (typically PDF) uploaded to `input/`
- Documents usually contain: step-by-step instructions, annotated screenshots, callout boxes, notes/warnings, and FAQs

## Outputs

Up to three files saved to `output/`, named after the source document:

| File | Purpose | When to Generate |
|------|---------|-----------------|
| `[Title] - LLM Readable.md` | Structured Markdown — primary knowledge base format | Always |
| `[Title] - LLM Readable.pdf` | PDF version of the Markdown — for sharing, printing, and archival | Always (generated from the .md) |
| `[Title] - LLM Readable.json` | Structured JSON — for programmatic/API consumption | When user requests JSON, or when both formats are requested |
| `[Title] - Eval.json` | Eval questions + expected answers + scoring rubrics | When user requests eval/test questions |

---

## CRITICAL: Zero-Hallucination Rule

**Every fact in the output must be directly traceable to explicitly visible content in the source PDF.** This is the single most important rule in this skill. Violations produce confidently wrong knowledge bases that poison downstream LLM answers.

### What this means in practice:

- **NEVER generate from memory or general impressions.** After reading a page, do not close it and write from recall. Extract verbatim first, then format.
- **NEVER fill gaps with domain knowledge.** Do not use your knowledge of Epic, EHR systems, healthcare workflows, or any application to infer what "should" be on a screen. Only describe what IS on the screen.
- **NEVER assume the document's purpose.** Extract the stated purpose verbatim. A "Downtime Patient Station" could be a lookup tool, a backlogging tool, or something else entirely — read what the document says it is.
- **NEVER fabricate UI elements.** If a screenshot shows 4 buttons, describe 4 buttons — not 8 because "applications like this usually have more."
- **NEVER invent warnings, notes, FAQs, or tips.** Only include these if they are explicitly present in the source document.
- **If you cannot clearly read a label, value, or element**, mark it as `[unreadable]` rather than guessing.

---

## Process

### Phase 1: Page-by-Page Verbatim Extraction

Read each page of the source PDF **individually as a separate image request** and capture a verbatim extraction before moving to the next page. Do NOT read all pages at once and generate from a general impression.

For each page, extract and write down:

1. **Exact text content** — copy every heading, paragraph, step instruction, sub-step, note, warning, and footer text verbatim as it appears on the page. Do not paraphrase.
2. **Screenshot inventory** — for each screenshot on the page, record:
   - Every field label and its current value (or note if empty) — **exact text only**
   - Every button label visible — **exact text only**
   - Every tab name visible — **exact text only**
   - Every table column header and visible row data — **exact text only**
   - Every dropdown and any visible options
   - Every annotation (arrows, circles, highlights, numbered callouts) and exactly what they point to — **see Annotation Priority Rule below**
   - Every status indicator, checkbox, icon, or visual state
   - **STOP here.** Do not add elements you "expect" to see but cannot explicitly read on the screen.
3. **Mark gaps honestly** — if any text or label is partially obscured, blurry, or unreadable, write `[unreadable]`. Do not guess.

#### Annotation Priority Rule

**Red boxes, arrows, circles, and highlights are the training author's voice.** They tell you what matters most and — critically — what BOUNDARIES define a group of related elements. Pay extremely close attention to:

- **What is INSIDE a red box vs. OUTSIDE it.** A red box around 4 of 9 buttons means only those 4 buttons are the relevant set for that step. The other 5 are separate, unrelated controls. Do not lump them together.
- **What an arrow points TO specifically.** An arrow to a single field means that field is the focus — not the entire dialog.
- **What is highlighted vs. what is adjacent.** A highlighted tab among 5 tabs means that tab is active/selected — the others are context, not the subject.

When the output describes annotated elements, **explicitly state the boundary**: which items are inside the annotation and which are outside. This distinction directly affects downstream ENUM definitions and prevents LLMs from treating adjacent-but-unrelated elements as part of the same set.

**Save the raw extraction** to `working/[document name] - Raw Extraction.md` before proceeding to Phase 2. This file is the single source of truth for all subsequent generation. If a fact is not in the raw extraction, it does not go in the output.

**After extracting all pages**, inventory the content types found:
- Procedural steps (numbered or sequential)
- Screenshots and annotated images
- Notes, warnings, important callouts
- Conditional guidance (if/then scenarios)
- Tables, field descriptions, dropdown values
- FAQs
- Headers, footers, revision metadata

**Ask clarifying questions** before generating if:
- The output format is ambiguous (offer: Markdown, JSON, PDF, or multiple)
- The audience is unclear (LLM-only vs. both LLM and human)
- Multiple documents are uploaded and processing order matters

### Phase 2: Generate LLM-Readable Content

Generate ONLY from the raw extraction file. If something is not in the extraction, it does not go in the output.

Follow these structural rules for all output formats:

#### Document Header
- Title, source organization, document type (tip sheet, SOP, etc.)
- Audience, system/application name, version, revision date
- Conversion note explaining that images have been replaced with textual descriptions

#### Purpose / Overview Section
- What the document teaches — **use the document's own words**, not a rewritten interpretation
- Prerequisites or required information (only if explicitly stated in the source)
- Important notes and warnings (preserved verbatim from source — do not add your own)

#### Enumerations Section

Include an **Enumerations (Constrained Value Sets)** section near the top of the document, after metadata and before the workflow. This section lists every finite, closed set of values found during extraction. ENUMs reduce downstream hallucination by constraining LLM responses to explicitly documented options.

**When to create an ENUM:**
- A screenshot shows a fixed set of buttons, tabs, dropdown options, or table columns
- A red box or annotation highlights a specific subset of elements — the ENUM is the subset INSIDE the annotation, not the full row
- A step lists specific options the user must choose from
- A dialog has a defined set of fields

**ENUM format:**

```
ENUM enum_name:
  - Value 1
  - Value 2
  - Value 3
```

**Rules:**
- Values must be **verbatim** from the source — exact spelling, exact capitalization
- If a set is known to be partial (e.g., only sample data rows visible), mark it `[partial]` and add a note
- If a red box defines the boundary of a set, note that in the ENUM description (e.g., "highlighted by red box in Step 5 screenshot")
- Group related ENUMs logically (e.g., all search-related ENUMs together)

#### Workflow Summary
- Group steps into logical phases with labels, step ranges, and one-sentence descriptions
- Present as a table for quick orientation

#### Step-by-Step Instructions
For each step:
- **Step number and action** — use the document's own wording for the action
- **Details** — additional context, sub-steps, or tips (only when present in source)
- **Conditional guidance** — scenario-specific instructions only if stated in the source
- **Screenshot descriptions** — see Image Description Rules below
- **ENUM references** — when a step involves choosing from a finite set, reference the ENUM by name (e.g., "see `ENUM encounter_type`")

#### FAQs (if present in source)
- Preserve exact Q&A pairs from the source document
- Add context from screenshot descriptions if a FAQ references a UI element
- **Do NOT generate FAQs that are not in the source document**

### Phase 3: Image Description Rules

This is the most critical phase. Every screenshot or annotated image in the source document must be converted to a detailed textual description. Follow these rules:

1. **Label each description** with a clear title: `[Screenshot: Descriptive Title]`
2. **Describe the full screen context** — what activity/dialog is shown, which patient/record, what part of the workflow
3. **List ONLY explicitly visible UI elements** — nothing inferred, nothing assumed:
   - Field names and their current values (populated or empty) — **exact text only**
   - Button labels and their states (enabled, disabled, highlighted) — **exact text only**
   - Dropdown options (list only options that are actually visible on screen)
   - Tab names, sidebar content, navigation elements — **exact text only**
   - Table columns and visible row data — **exact text only**
   - Warning icons, status indicators, checkboxes
   - **If you cannot read a label clearly, write `[unreadable]` — do not guess**
4. **Describe all visual annotations with boundary precision:**
   - Red boxes — state exactly which elements are INSIDE the box and which are OUTSIDE
   - Red arrows — note what they point to specifically
   - Highlighted/colored elements — note the color and what is highlighted
   - Callout boxes or numbered labels overlaid on the screenshot
   - **The annotation boundary defines the relevant set.** If a red box contains 4 of 9 buttons, only those 4 are the subject. The other 5 must be described separately as "outside the red box."
5. **Add a key insight** summarizing why this screenshot matters for the step it accompanies
6. **Use structured sub-elements** — break complex screenshots into labeled regions (e.g., "Left sidebar", "Main content area", "Bottom action bar") so the description is scannable

#### Example Image Description (Markdown)

```markdown
> **Screenshot: Patient Encounter Creation Dialog**
>
> The screenshot shows the **Patient Encounter Creation** dialog with two side-by-side examples.
>
> - **Left example — Internal provider:** Provider: GEARHARD, THOMAS E. Department: WMG FM/IM TLMC.
> - **Right example — External provider:** Provider: PROVIDER NOT IN SYSTEM. Department: WS ADULT MEDICINE SVS.
>
> *Key insight:* The left shows standard entry; the right shows the fallback for external providers without system privileges.
```

#### Example Image Description (JSON)

```json
{
  "title": "Patient Encounter Creation Dialog",
  "description": "The screenshot shows the Patient Encounter Creation dialog with two side-by-side examples.",
  "visual_elements": [
    {
      "label": "Left example — Internal provider",
      "details": "Provider: GEARHARD, THOMAS E. Department: WMG FM/IM TLMC."
    },
    {
      "label": "Right example — External provider",
      "details": "Provider: PROVIDER NOT IN SYSTEM. Department: WS ADULT MEDICINE SVS."
    }
  ],
  "key_insight": "The left shows standard entry; the right shows the fallback for external providers without system privileges."
}
```

### Phase 4: Verification Pass (MANDATORY)

After generating the LLM-readable output, **re-read each source page as an image one more time** and cross-check the output against the actual content:

1. **Step-level check:** For every step in the output, confirm the step number, action text, and sub-steps match the source. Flag any step that was reworded, added, or omitted.
2. **Screenshot-level check:** For every screenshot description in the output, re-examine the source screenshot and confirm:
   - Every field name in the output actually appears in the screenshot
   - Every button label in the output actually appears in the screenshot
   - Every table column in the output actually appears in the screenshot
   - Every value (patient name, dates, statuses) in the output matches the screenshot
   - No UI elements in the output are absent from the screenshot
3. **Annotation boundary check:** For every red box, arrow, or highlight described in the output, re-examine the screenshot and confirm:
   - The elements listed as INSIDE the annotation are actually inside it
   - The elements listed as OUTSIDE are actually outside it
   - No adjacent elements have been incorrectly grouped into an annotated set
4. **ENUM check:** For every ENUM in the output, confirm:
   - All values are verbatim from the source (exact spelling, capitalization)
   - No values have been added that are not visible in the source
   - Partial sets are marked `[partial]`
   - Annotation-bounded ENUMs match the annotation boundaries exactly
5. **No-additions check:** Confirm the output does not contain any warnings, notes, FAQs, tips, or contextual explanations that are not in the source document.
6. **Fix or flag** any discrepancy before delivering.

### Phase 5: Generate Eval Questions (When Requested)

Generate **20 questions** across **5 categories** (4 questions each):

| Category | What It Tests | Question Style |
|----------|---------------|----------------|
| **Factual Recall** | Can the LLM extract specific facts (names, values, counts) from the content? | "What is…", "How many…", "What are the three…" |
| **Procedural Reasoning** | Can it follow and sequence the workflow correctly? | "What is the next step after…", "Put these in order…" |
| **Conditional / Scenario-Based** | Can it apply the right guidance when conditions change? | "If [scenario], what should the user do?" |
| **UI / Screenshot Comprehension** | Can it reason over the textual screenshot descriptions as if seeing the actual UI? | "What options are in the dropdown…", "What is highlighted…" |
| **Interpretation / Reasoning** | Can it infer *why* things work the way they do? | "Why is X set to Y…", "What would happen if the user skipped…" |

#### Eval Structure (JSON)

```json
{
  "eval_metadata": {
    "title": "[Document Title] — LLM Comprehension Eval",
    "source_document": "[filename].md",
    "total_questions": 20,
    "categories": ["Factual Recall", "Procedural Reasoning", "Conditional / Scenario-Based", "UI / Screenshot Comprehension", "Interpretation / Reasoning"],
    "scoring": {
      "per_question_max": 3,
      "total_max": 60,
      "levels": {
        "3": "Full credit — all required elements present, accurate, no hallucination",
        "2": "Partial credit — most required elements present, minor omissions or imprecision",
        "1": "Minimal credit — at least one required element present but substantially incomplete",
        "0": "No credit — incorrect, hallucinated, or missing"
      },
      "pass_threshold_percent": 80,
      "pass_threshold_score": 48
    }
  },
  "eval_questions": [
    {
      "id": 1,
      "category": "Category Name",
      "question": "The question text",
      "expected_answer": "Full expected answer with reasoning",
      "required_elements": ["element 1", "element 2"],
      "scoring_rubric": {
        "3": "What earns full credit",
        "2": "What earns partial credit",
        "1": "What earns minimal credit",
        "0": "What earns no credit"
      }
    }
  ]
}
```

#### Eval Question Guidelines

- Every answer must be **fully derivable** from the LLM-readable content alone — no outside knowledge required
- **required_elements** should be checkable facts (names, values, sequences) that enable automated grading
- **scoring_rubric** entries should be specific enough for a human or LLM grader to score consistently
- Questions should range from simple lookup (Factual Recall) to multi-hop reasoning (Interpretation)
- At least 2 questions should require synthesizing information from screenshot descriptions
- At least 2 questions should require understanding conditional/branching logic in the steps
- At least 2 questions should test ENUM boundaries (e.g., "Is X a valid encounter type?" where X is outside the red box)
- Avoid yes/no questions — prefer questions that require explanation or enumeration

## Quality Checklist

Before delivering any output, verify:

- [ ] Every step from the source document is present (no gaps in numbering)
- [ ] Every screenshot/image has a corresponding textual description
- [ ] All field names, button labels, and dropdown values match the source exactly
- [ ] Conditional guidance (if/then scenarios) is preserved with both branches
- [ ] Notes and warnings are called out distinctly (not buried in body text)
- [ ] Metadata (version, date, author, copyright) is preserved
- [ ] The conversion note is present at the top explaining the image-to-text conversion
- [ ] Workflow summary groups steps into logical phases
- [ ] ENUMs are defined for every finite value set, with annotation boundaries respected
- [ ] Eval questions (if generated) cover all 5 categories with 4 questions each
- [ ] PDF version of the Markdown has been generated and confirmed in `output/`
- [ ] All output files are confirmed in `output/` via Glob before telling the user they're ready
- [ ] **ANTI-HALLUCINATION CHECK:** Re-read each source page and confirm the output contains ZERO facts, UI elements, warnings, notes, FAQs, or contextual details that are not explicitly present in the source document
- [ ] **ANNOTATION BOUNDARY CHECK:** Every red box, arrow, and highlight correctly distinguishes INSIDE vs. OUTSIDE elements — no adjacent items incorrectly grouped
- [ ] **RAW EXTRACTION EXISTS:** The `working/` directory contains the raw extraction file that was used as the source of truth for generation
