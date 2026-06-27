# Provenance Guard

A backend content-attribution system that classifies submitted text as human-written or AI-generated, returns a calibrated confidence score, displays a plain-language transparency label, and handles creator appeals.

---


## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [API Reference](#api-reference)
3. [Detection Signals](#detection-signals)
4. [Confidence Scoring](#confidence-scoring)
5. [Transparency Labels](#transparency-labels)
6. [Rate Limiting](#rate-limiting)
7. [Audit Log](#audit-log)
8. [Appeals Workflow](#appeals-workflow)
9. [Known Limitations](#known-limitations)
10. [Spec Reflection](#spec-reflection)
11. [AI Usage](#ai-usage)
12. [Running Locally](#running-locally)

---

## Architecture Overview

A submitted piece of text takes the following path through the system:

```
POST /submit  →  Request Validation  →  content_id assigned (UUID)
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   Signal 1 (LLM)        Signal 2 (Stylometrics)
   Groq llama-3.3-70b    sentence-length variance
   holistic assessment   type-token ratio
   → llm_score (0–1)     punctuation density
                         → stylo_score (0–1)
          └──────────┬──────────┘
                     ▼
             Confidence Scorer
             0.6×llm + 0.4×stylo → combined score
             > 0.70 → likely_ai
             0.40–0.70 → uncertain
             < 0.40 → likely_human
                     │
             Transparency Label Generator
             → human-readable label text
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
      Audit Log            JSON Response
  (audit_log.json)   content_id, attribution,
                     confidence, signals, label
```

**Appeal flow:**

```
POST /appeal  →  Look up content_id in audit log
                     │
            Update status → "under_review"
            Append appeal record with reasoning
                     │
            Return confirmation JSON
```

Every decision — classification, confidence score, individual signal scores, and any appeal — is captured in the structured audit log and retrievable at `GET /log`.

---

## API Reference

### `POST /submit`

Accepts a piece of text for attribution analysis.

**Request body:**
```json
{
  "text": "The text to analyze...",
  "creator_id": "creator-handle-or-id"
}
```

**Response:**
```json
{
  "content_id": "4775de21-1210-48bb-878d-e55a49e91da9",
  "attribution": "likely_ai",
  "confidence": 0.906,
  "llm_score": 0.95,
  "stylo_score": 0.8399,
  "label": "⚠️ Likely AI-Generated — Our analysis suggests this content was probably created using an AI writing tool (confidence: 91%). If you believe this is incorrect, the creator can submit an appeal.",
  "status": "classified"
}
```

**Rate limited:** 10 per minute, 100 per day per IP.

---

### `POST /appeal`

Allows a creator to contest a classification.

**Request body:**
```json
{
  "content_id": "aa163ae6-12fe-4930-b0e8-f7049febca1f",
  "creator_reasoning": "I wrote this myself from personal experience..."
}
```

**Response:**
```json
{
  "status": "appeal_received",
  "content_id": "aa163ae6-12fe-4930-b0e8-f7049febca1f",
  "message": "Your appeal has been logged and the content is now marked as 'under review'. A human reviewer will examine the original classification and your reasoning."
}
```

---

### `GET /log`

Returns recent audit log entries (default: last 50).

Optional query param: `?limit=N`

---

## Detection Signals

### Signal 1 — LLM-Based Classification (Groq, `llama-3.3-70b-versatile`)

**What it measures:** The semantic and stylistic coherence of the text as evaluated by a large language model. The LLM is prompted to score whether the text exhibits patterns typical of AI writing: consistent formal register, hedged phrasing ("it is important to note", "furthermore"), perfectly balanced paragraph structures, and absence of the idiosyncratic voice that human writers exhibit.

**Output:** A float in `[0.0, 1.0]`. The model is prompted to return only `{"ai_probability": 0.83}` which is parsed directly. Falls back to `0.5` on any API or parse error.

**Why it was chosen:** LLMs have strong priors about what AI-generated text looks like. This signal captures semantic and pragmatic patterns that rule-based heuristics cannot express — things like "this paragraph feels like a corporate communications template."

**What it misses:** Human writers who naturally write in a formal, structured style (academics, lawyers, careful non-native English speakers). It can also be fooled by AI text that has been lightly edited to introduce colloquialisms. It is sensitive to the LLM's own biases about what counts as "AI-like."

---

### Signal 2 — Stylometric Heuristics (Pure Python)

**What it measures:** Three structural statistics computed entirely from character and token patterns — no external API required:

1. **Sentence-length variance** — AI text tends toward consistent sentence lengths (low coefficient of variation). Human writing is more erratic. High variance → more human-like → lower AI score. Computed as coefficient of variation of per-sentence word counts, normalized to `[0.0, 1.0]`.

2. **Type-token ratio (TTR)** — ratio of unique word types to total tokens. Human casual writing clusters in a mid-range TTR (~0.55–0.65). Very high TTR in short formal text and very low TTR in repetitive text both suggest AI. The signal uses a V-shaped distance from the human TTR midpoint.

3. **Punctuation density / diversity** — AI text uses clean, standard punctuation (`.`, `,`, `:`, `;`, `?`). Human writing uses ellipses, em-dashes, exclamation marks, parentheses, and other unconventional patterns. More unusual punctuation types → more human-like → lower AI score.

**Output:** Each sub-metric produces a score in `[0.0, 1.0]`. The three scores are averaged (equal weight) into a single `stylo_score`.

**Why it was chosen:** These are structural properties that are independent of semantics. An LLM can produce semantically natural text but is less likely to naturally reproduce the messy distributional variance of a human typing casually. The two signals are genuinely orthogonal: one is semantic, one is structural.

**What it misses:** Formal human writing (academic papers, legal documents, well-edited essays) can score AI-like on all three metrics. Very short texts (< 3 sentences) give unreliable variance estimates. Poetry violates most assumptions — highly variable sentence length and punctuation usage make it unreliable for both signals.

---

## Confidence Scoring

### Combination formula

```
combined_score = 0.6 × llm_score + 0.4 × stylo_score
```

The LLM signal receives a higher weight (60%) because it evaluates the text holistically and captures semantic patterns. The stylometric signal (40%) provides independent structural evidence that anchors the score even when the LLM is uncertain.

### Threshold mapping

| Combined Score | Attribution    | Rationale |
|---|---|---|
| > 0.70         | `likely_ai`    | Both signals pointing meaningfully toward AI |
| 0.40 – 0.70    | `uncertain`    | Signals disagree or evidence is ambiguous |
| < 0.40         | `likely_human` | Evidence pointing toward human authorship |

### False positive asymmetry

The threshold for `likely_ai` is set at **0.70**, not 0.50. This deliberately biases toward `uncertain` in ambiguous cases. Misclassifying a human creator's work as AI-generated is a worse outcome than failing to catch AI content — the asymmetric threshold reflects that cost.

### Example submissions with real scores

**Example 1 — High-confidence AI (score: 0.906)**

Submitted text:
> "Artificial intelligence represents a transformative paradigm shift in modern society. It is important to note that while the benefits of AI are numerous, it is equally essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate to ensure responsible deployment."

| Signal | Score |
|---|---|
| LLM (`llm_score`) | 0.95 |
| Stylometric (`stylo_score`) | 0.8399 |
| **Combined** | **0.906 → `likely_ai`** |

Both signals agree strongly. The LLM detected characteristic hedged phrasing and uniform structure; stylometrics confirmed low sentence-length variance and standard punctuation only.

---

**Example 2 — Likely human (score: 0.3768)**

Submitted text:
> "ok so i finally tried that new ramen place downtown and honestly? underwhelming. the broth was fine but they put WAY too much sodium in it and i was thirsty for like three hours after. my friend got the spicy version and said it was better. probably wont go back unless someone drags me there"

| Signal | Score |
|---|---|
| LLM (`llm_score`) | 0.10 |
| Stylometric (`stylo_score`) | 0.7921 |
| **Combined** | **0.3768 → `likely_human`** |

The LLM strongly identifies this as human (casual register, unconventional capitalization, personal voice). The stylometric signal is higher because the sentence lengths are actually fairly consistent — showing the two signals can disagree, and the weighted combination reflects that disagreement. The LLM's strong signal pulls the result into `likely_human`.

---

**Example 3 — Uncertain (score: 0.4186)**

Submitted text:
> "I have been thinking a lot about remote work lately. There are genuine tradeoffs - flexibility and no commute on one side, isolation and blurred work-life boundaries on the other. Studies show productivity varies widely by individual and role type."

| Signal | Score |
|---|---|
| LLM (`llm_score`) | 0.20 |
| Stylometric (`stylo_score`) | 0.7466 |
| **Combined** | **0.4186 → `uncertain`** |

A borderline case: lightly edited or thoughtfully written text. The LLM reads a personal voice; stylometrics sees a formal-ish structure. The system correctly lands in `uncertain` and recommends treating the result with skepticism.

---

## Transparency Labels

Three label variants, each showing the exact text displayed to a reader on the platform:

**High-confidence AI (combined score > 0.70):**

> ⚠️ Likely AI-Generated — Our analysis suggests this content was probably created using an AI writing tool (confidence: 91%). If you believe this is incorrect, the creator can submit an appeal.

---

**Uncertain (combined score 0.40 – 0.70):**

> 🔍 Attribution Unclear — We couldn't confidently determine whether this content was written by a person or an AI tool (confidence: 42%). The result is inconclusive; treat it with appropriate skepticism.

---

**High-confidence human (combined score < 0.40):**

> ✓ Likely Human-Written — Our analysis suggests this content was written by a person (confidence: 62% human). Attribution is based on automated analysis and may not be perfect.

The `{score}%` in each label is the confidence in the stated conclusion:
- For `likely_ai`: shows the AI probability as a percentage
- For `likely_human`: shows `(1.0 - combined_score) × 100` as the human probability  
- For `uncertain`: shows the AI probability to convey how inconclusive the result is

---

## Rate Limiting

**Limits:** `10 per minute` and `100 per day` per IP address, applied to `POST /submit`.

**Reasoning:**

- A real human creator submits one piece of work at a time. Even an active user might submit 3–5 pieces per session, a few times per week. 10 per minute is generous for legitimate use while preventing scripted bulk scanning.
- 100 per day prevents a single actor from probing the detection system at scale to reverse-engineer the scoring thresholds.
- The per-minute burst limit targets automated submission scripts. A genuine human writer won't come close to it.
- False positives from shared IPs (e.g., a university writing lab) are acceptable — the limits are high enough that individual users on a shared network won't exhaust the pool.

**Observed rate-limit behavior** (12 rapid requests — exceeds 10/minute limit):

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

Requests 1–10 succeed; requests 11–12 return `429 Too Many Requests`.

---

## Audit Log

Every classification and every appeal is written to `audit_log.json` as a structured JSON array. Sample entries:

```json
[
  {
    "event": "classification",
    "content_id": "4775de21-1210-48bb-878d-e55a49e91da9",
    "creator_id": "test-user-1",
    "timestamp": "2026-06-27T17:22:28.768Z",
    "attribution": "likely_ai",
    "confidence": 0.906,
    "llm_score": 0.95,
    "stylo_score": 0.8399,
    "label": "⚠️ Likely AI-Generated — ...",
    "status": "classified",
    "text_preview": "Artificial intelligence represents a transformative paradigm shift..."
  },
  {
    "event": "classification",
    "content_id": "292ded9d-14fc-486f-bc55-2de87ab6cde6",
    "creator_id": "test-user-2",
    "timestamp": "2026-06-27T17:22:59.038Z",
    "attribution": "likely_human",
    "confidence": 0.3768,
    "llm_score": 0.1,
    "stylo_score": 0.7921,
    "label": "✓ Likely Human-Written — ...",
    "status": "classified",
    "text_preview": "ok so i finally tried that new ramen place downtown..."
  },
  {
    "event": "classification",
    "content_id": "aa163ae6-12fe-4930-b0e8-f7049febca1f",
    "creator_id": "test-user-4",
    "timestamp": "2026-06-27T17:23:18.370Z",
    "attribution": "uncertain",
    "confidence": 0.4186,
    "llm_score": 0.2,
    "stylo_score": 0.7466,
    "label": "🔍 Attribution Unclear — ...",
    "status": "under_review",
    "text_preview": "I have been thinking a lot about remote work lately..."
  },
  {
    "event": "appeal",
    "content_id": "aa163ae6-12fe-4930-b0e8-f7049febca1f",
    "timestamp": "2026-06-27T17:23:25.721Z",
    "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
    "status": "under_review"
  }
]
```

Fields captured per classification entry:
- `event`, `content_id`, `creator_id`, `timestamp`
- `attribution` (result), `confidence` (combined score)
- `llm_score`, `stylo_score` (individual signal scores)
- `label` (exact text shown to user)
- `status` (`classified` or `under_review`)
- `text_preview` (first 120 characters)

Fields captured per appeal entry:
- `event: "appeal"`, `content_id`, `timestamp`
- `appeal_reasoning`, `status: "under_review"`

Retrieve the log at any time: `GET /log` (or `GET /log?limit=10` for the 10 most recent).

---

## Appeals Workflow

Any creator who has a `content_id` from a prior submission can file an appeal at `POST /appeal`.

**What the appeal captures:**
- The `content_id` linking back to the original classification
- The creator's free-text reasoning

**What the system does:**
1. Looks up the `content_id` in the audit log — returns `404` if not found
2. Updates the classification entry's `status` from `classified` to `under_review`
3. Appends a new `event: "appeal"` record to the log with the reasoning and timestamp
4. Returns a confirmation JSON

**What a human reviewer sees:**
Querying `GET /log` surfaces the original entry (including both signal scores, the confidence, and the label that was shown) alongside the appeal record (the creator's stated reasoning). Everything needed for a manual judgment is in one place.

**No automated re-classification by design:** Automated re-classification would create an adversarial feedback loop — creators could learn exactly what reasoning text flips the system. Appeals are routed to human review only.

---

## Known Limitations

**Non-native English speakers writing formally.** A careful non-native English speaker whose writing is grammatically precise, uses formal vocabulary, and avoids colloquialisms will score AI-like on both signals. The formal register triggers the LLM signal; consistent sentence structure and standard punctuation trigger the stylometric signal. This is the most significant false-positive risk in the current system. The asymmetric threshold (0.70 for `likely_ai`) and the appeal workflow exist specifically for this scenario, but they don't eliminate the problem.

**Poetry and very short texts.** The sentence-length variance metric requires multiple sentences to compute a meaningful coefficient of variation. A haiku, a single-sentence excerpt, or a paragraph of poetry will produce unreliable stylometric scores. Poetry also violates assumptions about punctuation (deliberate fragmentation, mid-line breaks) and may have intentionally constrained vocabulary. Both signals are less reliable on very short or structurally unconventional text.

**Heavily edited AI output.** A creator who prompts an AI, then substantially rewrites the output to add personal anecdotes, introduce colloquial phrasing, and vary sentence lengths will likely score in the `uncertain` band. This is probably appropriate behavior — the content is genuinely collaborative — but the system cannot distinguish "heavily edited AI" from "human writing in a careful style."

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants explicitly in `planning.md` before touching any implementation code was the most directly useful part of the spec. It forced a concrete decision about what "uncertainty" means to a non-technical reader — which in turn determined the threshold values (0.40 and 0.70) and the asymmetric framing of the human confidence percentage. Without that, the thresholds would have been arbitrary numbers.

**One way implementation diverged from the spec:** The stylometric signal's type-token ratio (TTR) sub-metric was originally specified as a simple measure — high TTR → more AI-like. During testing it became clear this was wrong: human casual writing often has high TTR (varied vocabulary in short bursts), while some AI text has lower TTR due to repetitive filler phrases. The implementation was revised to a V-shaped distance from a human midpoint (~0.60) rather than a linear mapping. This is a more accurate model but deviates from the original spec description.

---

## AI Usage

**Instance 1 — Flask app skeleton and LLM signal function**

I provided the Detection Signals section from `planning.md` and the architecture diagram to Claude, and asked it to generate the Flask app skeleton with `POST /submit` stub and the `classify_with_llm()` function. The generated function returned a plain-text response from Groq rather than parsing structured JSON. I revised it to use a strict prompt instructing the model to return only `{"ai_probability": float}`, added a regex fallback for minor model non-compliance, and set `temperature=0.1` (the generated code used the default temperature, which introduced score variability on identical inputs that made calibration harder).

**Instance 2 — Confidence scoring logic and transparency label generator**

I provided the Uncertainty Representation and Transparency Label sections from `planning.md` and asked Claude to generate `compute_confidence()` and `generate_label()`. The generated `generate_label()` showed the raw `combined_score` percentage for all three variants — which was confusing for the `likely_human` case (a confidence of 0.35 reading as "35% confident" doesn't convey human authorship). I revised the label generator so that `likely_human` displays the human probability (`100 - combined_score × 100`) instead, which matches how the label's claim is worded.

---

## Running Locally

```bash
# 1. Clone and enter the repo
git clone <your-repo-url>
cd Provenance-Guard

# 2. Create and activate virtual environment
python -m venv .venv
# Windows:
.venv\Scripts\activate
# Mac/Linux:
source .venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Create .env with your Groq API key
echo "GROQ_API_KEY=your_key_here" > .env

# 5. Run the server
python app.py
# Server starts at http://localhost:5000

# 6. Test a submission
curl -s -X POST http://localhost:5000/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "It is important to note that AI represents a paradigm shift.", "creator_id": "demo-user"}' \
  | python -m json.tool

# 7. View the audit log
curl -s http://localhost:5000/log | python -m json.tool
```

**Dependencies** (see `requirements.txt`):
- `flask>=3.0.0`
- `flask-limiter>=3.5.0`
- `groq==0.15.0`
- `python-dotenv==1.0.1`
