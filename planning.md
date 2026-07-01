# Provenance Guard — planning.md

> Written before any implementation code.
> Updated before any stretch features.

---

## Overview

Provenance Guard is a backend API system that classifies text
as human-written or AI-generated, scores confidence in that classification,
surfaces a transparency label to users, and handles appeals from creators who
believe they've been misclassified.

---

## Architecture

### Submission Flow

SUBMISSION FLOW:
================

POST /submit (text, creator_id)
        │
        ▼
┌─────────────────────┐
│   Signal 1 (Groq)   │ → llm_score (0.0–1.0)
│   LLM Assessment    │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│   Signal 2          │ → stylometric_score (0.0–1.0)
│   Stylometrics      │   (sentence variance,
│   (pure Python)     │    TTR, punctuation density)
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Confidence Scoring │ → combined_score (0.0–1.0)
│  (weighted average) │   e.g. 0.6*llm + 0.4*stylo
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ Transparency Label  │ → label text (one of 3 variants)
│ Generator           │   based on score thresholds
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│    Audit Log        │ → writes structured JSON entry
│    (JSON file)      │   (content_id, scores, label,
└─────────────────────┘    timestamp, status)
        │
        ▼
JSON Response to client
(content_id, attribution, confidence, label)


APPEAL FLOW:
============

POST /appeal (content_id, creator_reasoning)
        │
        ▼
Look up content_id in audit log
        │
        ▼
Update status → "under_review"
        │
        ▼
Append appeal entry to audit log
(original decision + appeal_reasoning)
        │
        ▼
JSON Response: appeal received confirmation

### Architecture Narrative

A submitted piece of text passes through two independent detection signals,
an LLM-based semantic assessment and a stylometric structural analysis
whose outputs are combined into a single weighted confidence score. That score
maps to one of three transparency label variants, which is returned to the
client alongside the raw score and a unique content ID. Every submission
writes a structured entry to a persistent JSON audit log. If a creator
disputes the classification, they POST to /appeal with their content_id and
reasoning; the system updates the entry's status to "under_review" and logs
the appeal alongside the original decision.

---

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence whether the text
reads as human or AI-generated based on holistic properties like tone,
sentence variety, naturalness of expression, and overall writing feel.

**Output format:** A float between 0.0 and 1.0, where 1.0 means high
confidence the text is AI-generated and 0.0 means high confidence it is
human-written. The LLM is prompted to return only a numeric score.

**Why this signal:** LLMs are well-suited to holistic stylistic assessment
because they were trained on both human and AI text and can recognize subtle
patterns, overly formal phrasing, uniform sentence rhythm, absence of
personal voice, that are hard to capture with rules.

**What it misses:** The LLM may be biased toward flagging formal or
technically precise human writing as AI-generated. It also cannot reliably
detect lightly edited AI output where a human has introduced intentional
irregularities.

---

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Measurable structural properties that differ
statistically between human and AI writing. Computes three metrics:

- **Sentence length variance:** AI text tends to have consistent sentence
  lengths; human writing varies more. Low variance → higher AI score.
- **Type-token ratio (TTR):** Vocabulary diversity - unique words divided
  by total words. AI text reuses vocabulary more consistently; human writing
  is more diverse. Lower TTR → higher AI score.
- **Punctuation density:** Humans use punctuation more irregularly. Very
  uniform or low punctuation density → higher AI score.

**Output format:** Each metric is normalized to a 0.0–1.0 scale (where 1.0
means high likelihood of AI). The three metric scores are averaged into a
single stylometric_score float.

**Why this signal:** Stylometrics are computationally cheap, fully
transparent, and entirely independent of the LLM - they measure structural
properties the LLM doesn't explicitly analyze. This independence makes the
combined signal more informative than either alone.

**What it misses:** Formal human writing (academic papers, legal documents)
naturally has low sentence variance and low TTR, which may produce false
positives. Very short texts (under ~50 words) don't have enough data for
reliable metric calculation.

---

## Confidence Scoring and Uncertainty

**Combining signals:**
combined_score = (0.6 * llm_score) + (0.4 * stylometric_score)

The LLM signal receives higher weight (60%) because it captures holistic
semantic properties that are more diagnostically useful than structural
heuristics alone. The stylometric signal (40%) provides an independent
structural check.

**What a score of 0.6 means:** The system has moderate evidence that the
text is AI-generated but is not confident. Both signals are pointing
somewhat toward AI but neither strongly. A score of 0.6 produces the
"uncertain" label and the system declines to make a strong attribution
claim.

**Score thresholds:**

| Score range | Attribution | Label variant |
|-------------|-------------|---------------|
| 0.70 – 1.00 | likely_ai | High-confidence AI |
| 0.40 – 0.69 | uncertain | Uncertain |
| 0.00 – 0.39 | likely_human | High-confidence human |

**Why these thresholds:** The asymmetry reflects the cost of false positives.
Mislabeling a human writer's work as AI-generated is worse than missing an
AI submission. The "likely_human" threshold is set conservatively at 0.39
so the system only asserts human authorship when both signals agree strongly.
The "uncertain" band is wide (0.40–0.69) to capture all genuinely ambiguous
cases rather than forcing a binary verdict.

---

## Transparency Label Variants

### High-confidence AI (combined_score ≥ 0.70)
⚠️ This content was likely generated by AI.
Our system analyzed the writing style and found patterns strongly associated
with AI-generated text (confidence: {score}%).
If you are the human author of this work, you may submit an appeal using
your content ID: {content_id}

### Uncertain (combined_score 0.40–0.69)
❓ Our system could not confidently determine whether this content was
written by a human or generated by AI (confidence: {score}%).
This label reflects genuine uncertainty — not a finding of AI generation.
If you believe this label is inaccurate, you may submit an appeal using
your content ID: {content_id}

### High-confidence human (combined_score ≤ 0.39)
✅ This content appears to have been written by a human.
Our system found writing patterns consistent with human authorship
(confidence: {score}%).
Content ID: {content_id}

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted content and received a
content_id. No authentication is required in this implementation.

**What they provide:**
- `content_id` (str): The ID returned by the original /submit response
- `creator_reasoning` (str): The creator's explanation for why the
  classification is incorrect

**What the system does when an appeal is received:**
1. Looks up the content_id in the audit log
2. If not found: returns a 404 error with a clear message
3. If found: updates the entry's status from "classified" to "under_review"
4. Appends the appeal_reasoning and appeal_timestamp to the log entry
5. Returns a confirmation response with the content_id and new status

**What a human reviewer would see in the appeal queue:**
When querying GET /log, appeal entries show the original attribution result,
confidence score, both signal scores, the creator's reasoning, and the
timestamp of the appeal with everything needed to make a review decision.

**Automated re-classification:** Not implemented. Appeals require human
review.

---

## Anticipated Edge Cases

### Edge case 1: Formal human writing flagged as AI
Academic papers, legal documents, and technical writing naturally have low
sentence length variance, consistent vocabulary, and formal tone, which are all
properties the stylometric signal associates with AI. A law professor
submitting their own paper could receive a high AI confidence score.

**Mitigation:** The wide "uncertain" band (0.40–0.69) catches many of these
cases. The appeals workflow provides a path for misclassified creators.
Documented as a known limitation in the README.

### Edge case 2: Very short text (under 50 words)
Short texts don't provide enough data for reliable stylometric calculation.
Sentence length variance becomes meaningless with 2–3 sentences, and TTR
is unreliable for short texts because all words tend to be unique.

**Mitigation:** For texts under 50 words, the stylometric signal will be
downweighted or flagged as low-confidence. The system will rely more heavily
on the LLM signal for short submissions.

### Edge case 3: Lightly edited AI output
A user who takes AI-generated text and makes small manual edits (changing
a few words, adding personal anecdotes) may produce text that reads as
human to both signals.

**Mitigation:** This is an acknowledged limitation of current detection
technology. The system is designed to acknowledge uncertainty rather than
force binary verdicts, and the appeals workflow exists precisely for
contested cases.

---

## Rate Limiting

**Chosen limits:** 10 requests per minute, 100 requests per day per IP address.

**Reasoning:**
- A legitimate creator submitting their own work would rarely need more than
  a few submissions per session — 10 per minute is generous for normal use.
- 100 per day prevents scripted flooding while accommodating a power user
  who submits many pieces in one day.
- These limits protect the Groq API quota (the LLM call is the expensive
  operation) and prevent adversarial probing of the detection system.

---

## Audit Log Structure

Each log entry is a JSON object with the following fields:

```json
{
  "content_id": "uuid string",
  "creator_id": "string provided by submitter",
  "timestamp": "ISO 8601 datetime string",
  "attribution": "likely_ai | uncertain | likely_human",
  "confidence": 0.0,
  "llm_score": 0.0,
  "stylometric_score": 0.0,
  "label_text": "full transparency label string",
  "status": "classified | under_review",
  "appeal_reasoning": null,
  "appeal_timestamp": null
}
```

Appeal fields (`appeal_reasoning`, `appeal_timestamp`) are `null` until an
appeal is filed, then populated with the creator's reasoning and timestamp.

---

## API Endpoints

| Endpoint | Method | Input | Output |
|----------|--------|-------|--------|
| /submit | POST | text (str), creator_id (str) | content_id, attribution, confidence, label |
| /appeal | POST | content_id (str), creator_reasoning (str) | confirmation, status |
| /log | GET | none | list of recent audit log entries |

---

## AI Tool Plan

### Milestone 3 - Submission endpoint + Signal 1
**Input to AI:** Detection signals section (Signal 1 description, output
format) + Architecture diagram (submission flow) + API endpoints table.

**What I'll ask for:** Flask app skeleton with POST /submit route stub +
the Groq LLM signal function that returns a float between 0.0 and 1.0 +
a basic JSON audit log writer.

**How I'll verify:** Call the LLM signal function directly with 3 test
inputs (clearly AI, clearly human, borderline) and confirm the scores vary
meaningfully before wiring into the endpoint. Check that /submit returns
content_id, attribution, confidence, and label fields.

---

### Milestone 4 - Signal 2 + Confidence scoring
**Input to AI:** Detection signals section (Signal 2 description, three
metrics) + Confidence scoring section (thresholds, weighting formula) +
Architecture diagram.

**What I'll ask for:** Stylometric heuristics function computing sentence
length variance, TTR, and punctuation density + confidence scoring function
combining both signals with 60/40 weighting.

**How I'll verify:** Test the stylometric function independently on the same
4 inputs used for Signal 1. Confirm scores vary between clearly AI and
clearly human text. Verify the combined score matches expected threshold
ranges for each test input.

---

### Milestone 5 - Production layer
**Input to AI:** Transparency label variants section (exact text of all
three variants, score thresholds) + Appeals workflow section + Rate limiting
section + Architecture diagram (appeal flow).

**What I'll ask for:** Label generation function mapping scores to label
text + POST /appeal endpoint + Flask-Limiter configuration with chosen
limits.

**How I'll verify:** Test all three label variants are reachable by
submitting inputs that produce different confidence levels. Test the appeal
endpoint with a real content_id and verify status updates to "under_review"
in GET /log output.