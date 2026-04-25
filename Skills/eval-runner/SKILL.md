# Eval Runner

## Purpose

Runs a structured eval (question set with expected answers and scoring rubrics) against LLM-readable content to measure how well an LLM can reason over that content. Produces a detailed scorecard with per-question grades, category breakdowns, and a pass/fail result.

This skill is **content-agnostic** — it works with any eval JSON + content file pair, regardless of subject matter (training docs, SOPs, policy documents, etc.).

## When to Use

Trigger this skill when the user:
- Asks to "run the eval", "test the eval", "score the eval", or "execute the eval"
- Says "test the questions against the content"
- Asks "how well does the LLM do on these questions"
- Mentions "eval" or "evaluation" in the context of running/testing (not generating)
- Wants to validate that LLM-readable content is complete and accurate enough for AI consumption

## Inputs

The skill needs exactly **two files**:

| File | What It Is | How to Find |
|------|-----------|-------------|
| **Content file** | The LLM-readable document (`.md` or `.json`) that the LLM will reason over | Search `output/` for `*LLM Readable*` files |
| **Eval file** | The eval question set (`.json`) with questions, expected answers, required elements, and scoring rubrics | Search `output/` for `*Eval*` files |

**If files are ambiguous:** If multiple content/eval file pairs exist in `output/`, ask the user which pair to run. If only one pair exists, use it automatically.

## Process

### Step 1: Load Files

1. Find the eval JSON and content file in `output/` (or paths the user specifies)
2. Parse the eval JSON to extract:
   - `eval_metadata` (scoring config, pass threshold)
   - `eval_questions` (the full question set)
3. Read the full content file into memory — this becomes the **sole context** for answering

### Step 2: Answer Each Question

For each question in the eval:

1. **Prompt construction:** Build a prompt that provides ONLY the content file as context, then asks the question. The answering LLM must reason **exclusively** from the provided content — no outside knowledge.

2. **Prompt template:**

```
You are answering questions about the following document. Use ONLY the information in this document to answer. If the document does not contain enough information to answer, say "Insufficient information in the document."

---
DOCUMENT:
{content_file_text}
---

QUESTION: {question_text}

Provide a complete answer, citing specific details from the document (field names, values, step numbers, UI elements) to support your response.
```

3. **Capture the LLM-generated answer** for each question.

### Step 3: Grade Each Answer

For each question, compare the LLM-generated answer against the eval criteria:

1. **Required elements check:** For each item in `required_elements`, determine if the generated answer contains that element (exact or semantically equivalent).
2. **Rubric matching:** Match the answer quality to the appropriate `scoring_rubric` level (3, 2, 1, or 0).
3. **Hallucination check:** Flag if the answer contains claims not present in or contradicted by the content file.

**Grading prompt template:**

```
You are an eval grader. Compare the GENERATED ANSWER against the EXPECTED ANSWER and SCORING RUBRIC.

QUESTION: {question_text}

EXPECTED ANSWER: {expected_answer}

REQUIRED ELEMENTS (must be present for full credit):
{required_elements_list}

SCORING RUBRIC:
- 3: {rubric_3}
- 2: {rubric_2}
- 1: {rubric_1}
- 0: {rubric_0}

GENERATED ANSWER: {generated_answer}

Evaluate the generated answer and return:
1. SCORE (0-3): Which rubric level best matches the generated answer
2. ELEMENTS_FOUND: Which required elements are present (list each with yes/no)
3. HALLUCINATIONS: Any claims in the generated answer not supported by the expected answer (list or "None")
4. JUSTIFICATION: One sentence explaining the score
```

### Step 4: Produce the Scorecard

Generate a scorecard with three sections:

#### 4a. Summary

| Metric | Value |
|--------|-------|
| Total Score | X / {total_max} |
| Percentage | X% |
| Pass/Fail | PASS or FAIL (threshold: {pass_threshold_percent}%) |
| Questions Scored 3 (Full) | N |
| Questions Scored 2 (Partial) | N |
| Questions Scored 1 (Minimal) | N |
| Questions Scored 0 (Miss) | N |
| Hallucinations Detected | N |

#### 4b. Category Breakdown

| Category | Score | Max | Percentage |
|----------|-------|-----|------------|
| Factual Recall | X | 12 | X% |
| Procedural Reasoning | X | 12 | X% |
| Conditional / Scenario-Based | X | 12 | X% |
| UI / Screenshot Comprehension | X | 12 | X% |
| Interpretation / Reasoning | X | 12 | X% |

#### 4c. Per-Question Detail

For each question, show:
- Question ID and category
- Score (0–3) with the matched rubric description
- Required elements: which were found vs. missed
- Hallucination flag (if any)
- Justification for the score

### Step 5: Save and Present Results

1. **Save the full scorecard** to `output/` as `[Document Title] - Eval Scorecard.json`
2. **Present a summary** to the user inline (Summary table + Category Breakdown table)
3. **Highlight problem areas:** If any category scores below 50%, call it out with the specific questions that missed
4. **Recommend fixes:** If the eval fails, suggest which sections of the content file need enrichment based on which questions scored 0 or 1

## Scorecard Output Schema (JSON)

```json
{
  "scorecard_metadata": {
    "eval_title": "string",
    "content_file": "string",
    "eval_file": "string",
    "run_timestamp": "ISO 8601",
    "total_score": 0,
    "total_max": 60,
    "percentage": 0.0,
    "pass_fail": "PASS | FAIL",
    "pass_threshold_percent": 80
  },
  "category_scores": {
    "Factual Recall": { "score": 0, "max": 12, "percentage": 0.0 },
    "Procedural Reasoning": { "score": 0, "max": 12, "percentage": 0.0 },
    "Conditional / Scenario-Based": { "score": 0, "max": 12, "percentage": 0.0 },
    "UI / Screenshot Comprehension": { "score": 0, "max": 12, "percentage": 0.0 },
    "Interpretation / Reasoning": { "score": 0, "max": 12, "percentage": 0.0 }
  },
  "question_results": [
    {
      "id": 1,
      "category": "string",
      "question": "string",
      "generated_answer": "string",
      "expected_answer": "string",
      "score": 0,
      "rubric_level_matched": "string",
      "required_elements": {
        "element_1": true,
        "element_2": false
      },
      "hallucinations": "None | description",
      "justification": "string"
    }
  ],
  "recommendations": [
    "string — suggested content improvements if eval failed"
  ]
}
```

## Execution Options

The user may request variations:

| Request | Behavior |
|---------|----------|
| "Run the eval" | Full run: answer all 20 questions, grade, scorecard |
| "Run just the factual recall questions" | Filter to one category only |
| "Run questions 5, 9, and 14" | Run a specific subset by ID |
| "Re-run the eval after I updated the content" | Full run against the updated content file, compare to previous scorecard if available |
| "Show me which questions failed last time" | Read the most recent scorecard JSON and filter to score 0–1 |

## Important Notes

- **Isolation:** The answering step must use ONLY the content file as context. The eval questions and expected answers must NOT be visible to the answering prompt — otherwise the LLM is grading itself with the answer key in hand.
- **Grading separation:** The grading step is a separate prompt that sees both the generated answer and the expected answer. This two-stage approach (blind answer → informed grade) prevents answer leakage.
- **Hallucination sensitivity:** Any claim in a generated answer that cannot be traced to the content file should be flagged, even if the claim happens to be factually correct. The eval tests comprehension of the provided content, not general knowledge.
- **Scorecard persistence:** Always save the scorecard JSON to `output/` so results can be compared across runs (e.g., before and after content improvements).
