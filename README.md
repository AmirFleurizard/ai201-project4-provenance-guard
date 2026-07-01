# ai201-project4-provenance-guard

# Provenance Guard

---

## Overview

Provenance Guard is a backend API system that classifies text
as human-written or AI-generated, scores confidence in that classification,
surfaces a transparency label to users, and handles appeals from creators who
believe they've been misclassified.

---

## Architecture

A submitted piece of text passes through two independent detection signals.
An LLM-based semantic assessment and a stylometric structural analysis
whose outputs are combined into a single weighted confidence score. That score
maps to one of three transparency label variants which is returned to the
client alongside the raw score and a unique content ID. Every submission
writes a structured entry to a persistent JSON audit log. If a creator
disputes the classification, they POST to /appeal with their content_id and
reasoning; the system updates the entry's status to "under_review" and logs
the appeal alongside the original decision.

### Submission Flow

SUBMISSION FLOW:
================

POST /submit (text, creator_id)
        │
        ▼
┌─────────────────────┐
│   Signal 1 (Groq)   │ -> llm_score (0.0–1.0)
│   LLM Assessment    │
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│   Signal 2          │ -> stylometric_score (0.0–1.0)
│   Stylometrics      │   (sentence variance,
│   (pure Python)     │    TTR, punctuation density)
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│  Confidence Scoring │ -> combined_score (0.0–1.0)
│  (weighted average) │   e.g. 0.6*llm + 0.4*stylo
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│ Transparency Label  │ -> label text (one of 3 variants)
│ Generator           │   based on score thresholds
└─────────────────────┘
        │
        ▼
┌─────────────────────┐
│    Audit Log        │ -> writes structured JSON entry
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
        ├── Not found -> 404 error
        │
        └── Found -> update status to "under_review"
                  -> append appeal_reasoning + appeal_timestamp
                  -> return confirmationn

---

## API Endpoints

| Endpoint | Method | Input | Output |
|----------|--------|-------|--------|
| `/submit` | POST | `text` (str), `creator_id` (str) | `content_id`, `attribution`, `confidence`, `label_text`, `signals` |
| `/appeal` | POST | `content_id` (str), `creator_reasoning` (str) | confirmation, `status` |
| `/log` | GET | none | list of recent audit log entries |

---

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Semantic and stylistic coherence whether the text
reads as human or AI-generated based on holistic properties like tone,
sentence variety, naturalness of expression, and overall writing feel.


**Why I chose it:** LLMs are well-suited to holistic stylistic assessment
because they were trained on both human and AI text and can recognize subtle
patterns — overly formal phrasing, uniform sentence rhythm, absence of
personal voice — that are hard to capture with structural rules.

**What it misses:** The LLM may flag formal or technically precise human
writing as AI-generated. It also cannot reliably detect lightly edited AI
output where a human has introduced intentional irregularities.

### Signal 2: Stylometric Heuristics (Pure Python)

**What it measures:** Measurable structural properties that differ
statistically between human and AI writing. Computes three metrics:

- **Sentence length variance:** AI text tends to have consistent sentence
  lengths; human writing varies more. Low variance -> higher AI score.
- **Type-token ratio (TTR):** Vocabulary diversity - unique words divided
  by total words. AI text reuses vocabulary more consistently; human writing
  is more diverse. Lower TTR -> higher AI score.
- **Punctuation density:** Humans use punctuation more irregularly. Very
  uniform or low punctuation density -> higher AI score.

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

## Confidence Scoring

**Combination formula:**
```
combined_score = (0.6 × llm_score) + (0.4 × stylometric_score)
```

The LLM signal receives higher weight (60%) because it captures holistic
semantic properties that are more diagnostically useful than structural
heuristics alone.

**Score thresholds:**

| Score range | Attribution | Label variant |
|-------------|-------------|---------------|
| 0.70 – 1.00 | `likely_ai` | High-confidence AI |
| 0.40 – 0.69 | `uncertain` | Uncertain |
| 0.00 – 0.39 | `likely_human` | High-confidence human |

The "uncertain" band is intentionally wide to avoid forcing a binary verdict
on genuinely ambiguous content. A false positive — labeling a human's work
as AI-generated — is worse than a false negative on a creative platform,
so the system requires strong agreement from both signals before asserting
`likely_ai`.

**Validation — two example submissions with noticeably different scores:**

*Example 1 — Clearly AI-generated text:*
```
"Artificial intelligence represents a transformative paradigm shift in modern
society. It is important to note that while the benefits of AI are numerous,
it is equally essential to consider the ethical implications..."
```
- LLM score: 0.80
- Stylometric score: 0.50
- Combined confidence: **0.68** -> `uncertain`

*Example 2 — Clearly human-written text:*
```
"ok so i finally tried that new ramen place downtown and honestly?
underwhelming. the broth was fine but they put WAY too much sodium in it..."
```
- LLM score: 0.20
- Stylometric score: 0.49
- Combined confidence: **0.32** -> `likely_human`

The two examples produce meaningfully different scores (0.68 vs 0.32),
demonstrating that the scoring function captures real variation rather than
returning a constant. Note that the AI example scores in the "uncertain"
range rather than "likely_ai" — this reflects the conservative threshold
design and the stylometric signal's moderate score on that text.

---

## Transparency Label Variants

All three label variants are shown below with exact text as displayed to users.

**High-confidence AI (confidence ≥ 0.70):**
```
⚠️ This content was likely generated by AI.

Our system analyzed the writing style and found patterns strongly associated
with AI-generated text (confidence: {score}%).

If you are the human author of this work, you may submit an appeal using
your content ID: {content_id}
```

**Uncertain (confidence 0.40–0.69):**
```
❓ Our system could not confidently determine whether this content was written
by a human or generated by AI (confidence: {score}%).

This label reflects genuine uncertainty — not a finding of AI generation.
If you believe this label is inaccurate, you may submit an appeal using
your content ID: {content_id}
```

**High-confidence human (confidence ≤ 0.39):**
```
✅ This content appears to have been written by a human.

Our system found writing patterns consistent with human authorship
(confidence: {score}%).

Content ID: {content_id}
```

---

## Rate Limiting

**Limits:** 10 requests per minute, 100 requests per day (per IP address).

**Reasoning:**
- A legitimate creator submitting their own work would rarely need more than
  a few submissions per session — 10 per minute is generous for normal use.
- 100 per day prevents scripted flooding while accommodating a power user
  who submits many pieces in one day.
- These limits protect the Groq API quota (the LLM call is the expensive
  operation) and prevent adversarial probing of the detection system.

**Rate limit behavior — verified output (12 rapid requests, limit is 10/min):**
```
200
200
200
200
200
200
200
200
200
200
429
429
```
Requests 11 and 12 correctly receive HTTP 429 (Too Many Requests).

---

## Audit Log

Every attribution decision is captured in `audit_log.json`. Each entry
contains:

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

**Sample log output (GET /log):**
```json
{
  "total_entries": 2,
  "entries": [
    {
      "content_id": "3e4c6ad9-85bc-420f-b494-cbc2ebd88603",
      "creator_id": "test-user-2",
      "timestamp": "2026-07-01T00:18:36.702437+00:00",
      "attribution": "likely_human",
      "confidence": 0.3159,
      "llm_score": 0.2,
      "stylometric_score": 0.4898,
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "content_id": "d13a47ff-f22a-4fc9-a1b9-d48a30ae01b2",
      "creator_id": "test-user-1",
      "timestamp": "2026-07-01T00:16:33.758681+00:00",
      "attribution": "uncertain",
      "confidence": 0.6815,
      "llm_score": 0.8,
      "stylometric_score": 0.5038,
      "status": "under_review",
      "appeal_reasoning": "I wrote this myself from personal experience as a
        researcher. My academic writing style may appear more formal than typical.",
      "appeal_timestamp": "2026-07-01T00:18:49.072121+00:00"
    }
  ]
}
```

---

## Appeals Workflow

Creators who believe they have been misclassified can submit an appeal via
`POST /appeal` with their `content_id` and `creator_reasoning`. The system:

1. Looks up the `content_id` in the audit log
2. If not found: returns a 404 error with a clear message
3. If found: updates the entry's status from `"classified"` to
   `"under_review"` and records the appeal reasoning and timestamp
4. Returns a confirmation response

Automated re-classification is not implemented — appeals require human
review. The GET /log endpoint surfaces all appeal details for a reviewer:
original attribution, confidence score, both signal scores, the creator's
reasoning, and the appeal timestamp.

**Example appeal request:**
```bash
curl -s -X POST http://localhost:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{
    "content_id": "d13a47ff-f22a-4fc9-a1b9-d48a30ae01b2",
    "creator_reasoning": "I wrote this myself from personal experience as a
      researcher. My academic writing style may appear more formal than typical."
  }'
```

**Example response:**
```json
{
  "appeal_timestamp": "2026-07-01T00:18:49.072121+00:00",
  "content_id": "d13a47ff-f22a-4fc9-a1b9-d48a30ae01b2",
  "message": "Your appeal has been received and is under review.",
  "status": "under_review"
}
```

---

## Known Limitations

**Formal human writing produces false positives.**
Academic papers, legal documents, and technical writing naturally have low
sentence length variance, consistent vocabulary, and formal tone — all
properties the stylometric signal associates with AI. A researcher or lawyer
submitting their own work could receive an `uncertain` or even `likely_ai`
classification. This is a structural limitation of stylometric detection
and is mitigated by the wide "uncertain" band and the appeals workflow,
but not eliminated.

**Very short texts are unreliable.**
Texts under approximately 50 words don't provide enough data for meaningful
stylometric analysis. With only 2–3 sentences, sentence length variance is
statistically meaningless and TTR is unreliable because all words tend to
be unique in short texts. For short submissions, the system falls back to
the LLM signal alone.

**Lightly edited AI output may not be detected.**
A user who takes AI-generated text and makes small manual edits —
changing a few words, adding personal details — may produce text that
scores low on both signals. Perfect AI detection is an unsolved problem;
this system is designed to acknowledge uncertainty honestly rather than
claim detection accuracy it cannot deliver.

---

## Spec Reflection

**One way the spec helped:** Writing out the exact text of all three label
variants in planning.md before building the label generator made
implementation straightforward — I had a concrete contract to implement
against rather than figuring out the copy and the code at the same time.
The thresholds (0.70 / 0.40 / 0.39) were also defined in planning.md and
transferred directly into the `get_attribution()` function with no
ambiguity.

**One way implementation diverged from the spec:** The planning.md spec
anticipated that clearly AI-generated text would score in the `likely_ai`
range (≥ 0.70). In testing, the AI text sample scored 0.68 — landing in
`uncertain` rather than `likely_ai`. This revealed that the stylometric
signal underscores clearly AI-generated text because the sample was short
enough that the metrics were borderline. In hindsight I would add a
minimum text length requirement (50+ words) before applying stylometrics,
and fall back to LLM-only scoring for short texts rather than pulling the
combined score down.

---

## AI Usage

**Instance 1 — Flask app skeleton and Signal 1:**
I gave Claude my detection signals section and architecture diagram from
planning.md and asked it to generate the Flask app skeleton with the
POST /submit route and the Groq LLM signal function. It produced a
complete app.py with the route structure, audit log helpers, and the
signal_llm() function. I reviewed the prompt it used for the LLM signal
and rewrote it to be more specific — the original version asked for a
classification label rather than a numeric score, which would have required
additional parsing logic. I also added the 0.5 fallback return value on
exception, which the generated code was missing.

**Instance 2 — Stylometric signal and confidence scoring:**
I gave Claude my Signal 2 description (sentence length variance, TTR,
punctuation density) and the confidence scoring formula (0.6/0.4 weighting)
from planning.md and asked it to implement signal_stylometric() and
compute_confidence(). The generated stylometric function computed the
metrics correctly but used arbitrary normalization ranges that didn't match
my planning.md reasoning. I revised the normalization for each metric to
match the documented ranges (std_dev capped at 15, TTR range 0.4–0.9) so
the scores would be interpretable against my spec rather than arbitrary
internal values.
