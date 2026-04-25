# Epic Tip Sheet Optimizer

Skills, processes, and artifacts — powered by **M365 Copilot Cowork** — for converting screenshot-heavy Epic training tip sheets into LLM-readable knowledge bases with structured eval generation and validation, so M365 Copilot agents can actually use them.

## Problem Statement

Healthcare organizations maintain thousands of Epic training tip sheets — step-by-step guides that teach staff how to perform clinical and operational workflows. These documents are heavily visual: nearly every step includes annotated screenshots with red boxes, arrows, and callout labels that carry critical contextual information.

**AI agents struggle with these documents.** Screenshots are opaque to LLMs. When a tip sheet says "click the highlighted button" and the only way to know *which* button is to read the red-boxed annotation in the screenshot, the agent is blind. Multiply this across thousands of tip sheets, and you have a knowledge base that is:

- **Unsearchable** — agents can't reason over images embedded in PDFs or Word docs
- **Inaccurate** — agents that attempt to answer from partial text hallucinate the missing visual context
- **Unmaintainable** — when Epic releases updates, there's no structured way to determine which tip sheets are impacted

This repo provides the **Copilot skills and processes** to solve this at scale.

## How It Works

The solution is a two-track process powered by **M365 Copilot Cowork** — which orchestrates document conversion, skill execution, and reasoning — combined with custom Copilot skills that handle ingestion, eval generation, and analysis.

![Tip Sheet End-to-End Process](Assets/TipSheetEndtoEnd.png)

### Primary Process: Convert Tip Sheets, Generate & Run Evals, Store for Copilot Agent

| Step | Action | Details |
|------|--------|---------|
| **1** | **Select Tip Sheet** | Choose an Epic tip sheet (Word/PDF source) for conversion |
| **2** | **Convert to LLM-Readable PDF** | Use M365 Cowork to convert the source document into an LLM-readable PDF |
| **3** | **Ingest & Structure Tip Sheet** | Run the **Training Content to Knowledge Base** skill — extracts steps and screenshots via OCR, producing a structured, LLM-readable tip sheet (Markdown and/or JSON) |
| **4** | **Run Eval Generation Skill** | Generate a 20-question eval set (JSON) with expected answers and scoring rubrics across 5 categories |
| **5** | **Validate Content Accuracy** | Run the **Eval Runner** skill — executes the eval against the structured content, producing a scorecard with pass/fail, category breakdowns, and per-question results |
| **6** | **Store in SharePoint** | Publish the validated tip sheet and eval data to a SharePoint library where M365 Copilot and/or WallE agents are grounded |

### Secondary & Related Process: Process What's New & NOVA Release Notes and Determine Impact to Tip Sheets

| Step | Action | Details |
|------|--------|--------|
| **1** | **Ingest Release Notes Sources** | Collect What's New updates and Epic NOVA release notes |
| **2** | **Convert to LLM-Readable using M365 Cowork** | Cowork converts release notes documents into PDFs optimized for LLM consumption |
| **3** | **Ingest with Release Notes Ingestion Skill** | Extract features, changes, fixes, and enhancements; identify impacted areas (workflows, screens, fields, functionality); normalize and structure content |
| **4** | **Reason Over Tip Sheets vs. Release Notes** | Use M365 Cowork to compare structured release notes against structured tip sheets — identify changes that impact existing tip sheets, gaps (new functionality not covered), and new tip sheets needed |
| **5** | **Output Impact Analysis** | Generate recommendations: tip sheets to update, new tip sheets needed, or no action — stored as an impact report (JSON/Report) in SharePoint |

### Accessible by M365 Copilot Agent

The M365 Copilot Agent is grounded in the SharePoint library and can:

- Answer questions about tip sheets
- Surface relevant steps and screenshots
- Explain features and workflows
- Support analysts, clinicians, and training teams

All responses are grounded in the stored, validated content.

### Key Enablers & Components

| Component | Role |
|-----------|------|
| **M365 Cowork** | Orchestrates document conversion, skill execution, and reasoning |
| **Skills (Custom)** | Consistent logic for ingestion, structure, eval generation, and analysis |
| **Evaluations** | Ensure quality, accuracy, and completeness of extracted content |
| **SharePoint** | System of record and source of truth for Copilot Agent grounding |
| **M365 Copilot Agent** | Delivers trusted, grounded answers using your organization's content |

## Repository Structure

```
├── Assets/                          # Process diagrams and visual assets
│   └── TipSheetEndtoEnd.png         # End-to-end process overview
├── Skills/                          # Copilot skills (SKILL.md files)
│   ├── training-content-to-knowledge-base/
│   │   └── SKILL.md                 # Converts tip sheets → LLM-readable content + evals
│   └── eval-runner/
│       └── SKILL.md                 # Runs evals against content, produces scorecards
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── LICENSE                          # MIT License
├── SECURITY.md
└── README.md
```

## Skills

### Training Content to Knowledge Base

Converts training tip sheets (typically PDFs with annotated screenshots) into structured, LLM-readable knowledge base content. Performs page-by-page verbatim extraction with strict zero-hallucination rules, generates ENUMs for constrained value sets, and respects annotation boundaries (what's inside vs. outside a red box). Optionally generates a 20-question eval set across 5 categories.

**[View Skill →](Skills/training-content-to-knowledge-base/SKILL.md)**

### Eval Runner

Runs a structured eval against LLM-readable content to measure how well an LLM can reason over it. Uses a two-stage approach (blind answer → informed grade) to prevent answer leakage. Produces a detailed scorecard with per-question grades, category breakdowns, hallucination flags, and pass/fail results.

**[View Skill →](Skills/eval-runner/SKILL.md)**

## Getting Started

1. **Clone this repo**
   ```bash
   git clone https://github.com/martycarreras-psnl/Epic-Tip-Sheet-Optimizer.git
   ```

2. **Install the skills** into your Copilot environment by copying the `Skills/` folder into your workspace's skill directory (e.g., `~/.copilot/skills/`)

3. **Prepare your tip sheet** — place the source PDF into an `input/` folder in your working directory

4. **Start a Cowork session** and use the prompts below to drive the end-to-end process

## Example Prompts

Use these prompts in M365 Copilot Cowork to trigger the skills. They're designed to follow the end-to-end process in sequence.

### Step 1 — Convert a Tip Sheet to LLM-Readable Content

> **"Convert this tip sheet to an LLM-readable knowledge base"**

Other ways to phrase it:
- *"Make this training document AI-friendly"*
- *"Ingest this tip sheet and produce a structured knowledge base"*
- *"Convert this job aid so an LLM can understand it"*
- *"Build a knowledge base from this training material"*

This triggers the **Training Content to Knowledge Base** skill. It will:
- Extract every page verbatim (text + screenshot descriptions)
- Generate ENUMs for constrained value sets (buttons, tabs, dropdowns)
- Produce a structured Markdown and/or JSON output in `output/`

### Step 2 — Generate Eval Questions

> **"Generate eval questions to test the LLM's ability to reason over this content"**

Other ways to phrase it:
- *"Create an eval for this tip sheet"*
- *"Generate test questions with expected answers and scoring rubrics"*
- *"Build an eval question set for the knowledge base content"*

This produces a 20-question eval (JSON) across 5 categories: Factual Recall, Procedural Reasoning, Conditional/Scenario-Based, UI/Screenshot Comprehension, and Interpretation/Reasoning.

### Step 3 — Run the Eval

> **"Run the eval"**

Other ways to phrase it:
- *"Test the eval against the content"*
- *"Score the eval"*
- *"How well does the LLM do on these questions?"*

This triggers the **Eval Runner** skill. It answers each question using *only* the content file (no answer key leakage), then grades each answer against the rubric. Output is a scorecard with pass/fail, category breakdowns, and per-question detail.

### Targeted & Follow-Up Prompts

Once you've run a full eval, you can drill into specific areas:

| Prompt | What It Does |
|--------|-------------|
| *"Run just the factual recall questions"* | Runs only one eval category |
| *"Run questions 5, 9, and 14"* | Runs a specific subset by question ID |
| *"Re-run the eval after I updated the content"* | Full re-run, compares to previous scorecard |
| *"Show me which questions failed last time"* | Filters the most recent scorecard to scores 0–1 |
| *"Which categories scored below 50%?"* | Surfaces weak areas in the content |

### End-to-End Prompt Sequence (Copy-Paste Ready)

For a complete tip sheet conversion, use these prompts in order:

```
1. "Convert this tip sheet to an LLM-readable knowledge base and also generate the eval questions"
2. "Run the eval"
3. "Which questions scored 0 or 1? What content needs to be improved?"
4. [Make the suggested improvements to the content]
5. "Re-run the eval after I updated the content"
```

Repeat steps 3–5 until the eval passes (≥80% threshold).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on submitting skills, improvements, and bug reports.

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for details.

## Security

For security reporting guidance, see [SECURITY.md](SECURITY.md).
