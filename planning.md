# Provenance Guard — Planning Document

## Architecture

### Architecture Narrative

A piece of submitted text enters the system at `POST /submit`. The endpoint validates the request, assigns a unique `content_id`, then passes the raw text to two completely independent detection signals running in sequence. Signal 1 (LLM-based via Groq) evaluates the text holistically — semantic coherence, phrasing patterns, and stylistic uniformity — and returns a probability score between 0 and 1 (1 = AI). Signal 2 (stylometric heuristics) measures three structural statistics about the text — sentence-length variance, type-token ratio, and punctuation density — and combines them into a second score between 0 and 1 (1 = AI-like). The confidence scorer then combines these two scores using a weighted average (LLM: 60%, stylometric: 40%) and maps the result to a human-readable transparency label. The label text, both signal scores, the combined confidence, and the attribution result are written to the structured audit log and returned to the caller in a single JSON response. An appeal is entirely separate: `POST /appeal` accepts a `content_id` and the creator's reasoning, updates the stored entry's status to `under_review`, appends an appeal record to the audit log, and returns confirmation. Nothing else in the pipeline changes on appeal — a human reviewer would inspect the log.

### Architecture Diagram

```
POST /submit
     │
     ▼
┌─────────────────────────────────────────────────────┐
│  Request Validation                                  │
│  • require `text` and `creator_id`                   │
│  • assign unique content_id (UUID)                   │
└────────────────────┬────────────────────────────────┘
                     │  raw text
          ┌──────────┴──────────┐
          ▼                     ▼
┌──────────────────┐  ┌──────────────────────────────┐
│  Signal 1        │  │  Signal 2                    │
│  LLM Classifier  │  │  Stylometric Heuristics      │
│  (Groq API)      │  │  • sentence-length variance  │
│  → llm_score     │  │  • type-token ratio          │
│    (0.0–1.0)     │  │  • punctuation density       │
└────────┬─────────┘  │  → stylo_score (0.0–1.0)    │
         │            └──────────────┬───────────────┘
         │                           │
         └──────────┬────────────────┘
                    ▼
        ┌───────────────────────┐
        │  Confidence Scorer    │
        │  combined =           │
        │   0.6×llm + 0.4×stylo │
        │                       │
        │  > 0.70  → likely_ai  │
        │  0.40–0.70 → uncertain│
        │  < 0.40  → likely_    │
        │             human     │
        └──────────┬────────────┘
                   │  (attribution, confidence)
                   ▼
        ┌───────────────────────┐
        │  Transparency Label   │
        │  Generator            │
        │  maps score → label   │
        │  text string          │
        └──────────┬────────────┘
                   │
          ┌────────┴────────┐
          ▼                 ▼
┌─────────────────┐  ┌──────────────────────────────┐
│  Audit Log      │  │  JSON Response to caller     │
│  (JSON file)    │  │  { content_id, attribution,  │
│  append entry   │  │    confidence, llm_score,    │
└─────────────────┘  │    stylo_score, label }      │
                     └──────────────────────────────┘


POST /appeal
     │
     ▼
┌────────────────────────────────────────────────────┐
│  Lookup content_id in audit log                    │
│  • 404 if not found                                │
│  • update status → "under_review"                  │
│  • append appeal record to log:                    │
│    { event: "appeal", appeal_reasoning, timestamp} │
└────────────────────┬───────────────────────────────┘
                     │
                     ▼
          ┌────────────────────┐
          │  Return            │
          │  { status: ok,     │
          │    content_id,     │
          │    message }       │
          └────────────────────┘
```

---

## Detection Signals

### Signal 1 — LLM-Based Classification (Groq, llama-3.3-70b-versatile)

**What it measures:** The semantic and stylistic coherence of the text as evaluated by a large language model. The LLM is prompted to assess whether the text exhibits patterns typical of AI writing: consistent formal register, hedged phrasing ("it is important to note"), perfectly balanced sentence structures, and absence of the idiosyncratic detours that human writers make.

**Output format:** A float between 0.0 and 1.0 where 1.0 means "almost certainly AI-generated." The model is instructed to return a JSON object `{"ai_probability": 0.83}` only — no prose — which is parsed directly.

**Why this signal captures something real:** LLMs trained on internet text have strong priors about what AI-generated text looks like because they've seen large amounts of it. A model can detect subtle coherence patterns that rule-based heuristics can't express.

**Blind spots:** The LLM signal can be fooled by human writers who naturally write in a formal, structured style (academics, lawyers, non-native English speakers whose writing is careful and precise). It is also potentially circular — a model trained on AI text assessing AI text. It has no knowledge of a specific creator's historical voice.

---

### Signal 2 — Stylometric Heuristics (Pure Python)

**What it measures:** Three structural statistics that differ systematically between human and AI writing:

1. **Sentence-length variance** — AI text tends toward consistent sentence lengths. Human writing is more erratic. A high variance score means more human-like irregularity (so we *invert* it for the AI score).
2. **Type-token ratio (TTR)** — the ratio of unique words to total words. AI text reuses vocabulary less unpredictably than humans do in casual writing but may be *higher* in formal AI text. We calibrate this specifically: very high TTR in short formal text is an AI signal.
3. **Punctuation density** — AI text tends toward clean, complete sentences with standard punctuation. Human casual writing drops commas, uses ellipses, exclamation marks, and unconventional punctuation patterns.

**Output format:** A float between 0.0 and 1.0 (1.0 = AI-like). Each sub-metric is computed, normalized, and combined with equal weights (1/3 each) into a single `stylo_score`.

**Why this signal captures something real:** These are structural properties of the text that don't depend on semantics at all. They capture consistency at the character/token level. An LLM can produce semantically natural text but is less likely to naturally reproduce the messy variance patterns of a human typing a casual piece.

**Blind spots:** Formal human writing (academic papers, legal documents, well-edited essays) can look very AI-like on stylometric grounds. Very short texts give unreliable variance estimates because there aren't enough sentences. Highly edited AI output where a human has deliberately introduced irregularities can fool the heuristic.

---

## Uncertainty Representation

**What 0.6 means:** A confidence score of 0.6 means the system has mild evidence pointing toward AI authorship but not enough to assert it confidently. Displayed to the user it reads as "Attribution Unclear" — the system is honest that it cannot make a reliable call.

**Mapping from raw signal outputs to calibrated score:**

The combined score is `0.6 × llm_score + 0.4 × stylo_score`. This weighting reflects that the LLM signal is generally more reliable (it sees the full semantic picture) but stylometrics provide independent structural evidence worth counting.

**Threshold boundaries:**

| Combined Score | Attribution | Rationale |
|---|---|---|
| > 0.70 | `likely_ai` | Both signals are in reasonable agreement and pointing toward AI |
| 0.40 – 0.70 | `uncertain` | Signals disagree or evidence is ambiguous; system acknowledges it doesn't know |
| < 0.40 | `likely_human` | Evidence points toward human authorship |

**False positive asymmetry:** The threshold for `likely_ai` is set at 0.70 (not 0.50) to reduce the risk of incorrectly labeling a human creator's work. This deliberately biases toward `uncertain` in ambiguous cases — the cost of a false positive (insulting a human creator) is higher than a false negative (not catching AI-generated content).

---

## Transparency Label Design

Three label variants, written in plain language for a non-technical reader:

**High-confidence AI (combined score > 0.70):**
> ⚠️ Likely AI-Generated — Our analysis suggests this content was probably created using an AI writing tool (confidence: {score}%). If you believe this is incorrect, the creator can submit an appeal.

**Uncertain (combined score 0.40–0.70):**
> 🔍 Attribution Unclear — We couldn't confidently determine whether this content was written by a person or an AI tool (confidence: {score}%). The result is inconclusive; treat it with appropriate skepticism.

**High-confidence human (combined score < 0.40):**
> ✓ Likely Human-Written — Our analysis suggests this content was written by a person (confidence: {score}% human). Attribution is based on automated analysis and may not be perfect.

The `{score}` in all three variants is the human-readable confidence in the stated conclusion — not the raw AI-probability. For `likely_ai`, it shows the AI probability as a percentage. For `likely_human`, it shows `100 - combined_score` as the human probability.

---

## Appeals Workflow

**Who can appeal:** Any creator with a `content_id` from a previous `/submit` response can file an appeal.

**What they provide:**
- `content_id` — the ID of the submission to contest
- `creator_reasoning` — free-text explanation (e.g., "I am a non-native English speaker and my writing style may appear formal")

**What the system does on appeal:**
1. Looks up the `content_id` in the audit log
2. Updates the entry's `status` field from `classified` to `under_review`
3. Appends a new audit log record with `event: "appeal"`, the `creator_reasoning`, and a timestamp
4. Returns a confirmation JSON to the caller

**What a human reviewer would see:** Querying `GET /log` returns all entries including the appeal record linked to the original classification by `content_id`. The reviewer can see the original signal scores, the confidence, the label that was shown, and the creator's stated reasoning — everything needed to make a manual judgment.

**No automated re-classification:** Appeals are intentionally human-reviewed. Automated re-classification would create an adversarial loop where creators learn the exact inputs that flip the system.

---

## Anticipated Edge Cases

**Edge case 1 — Highly edited AI output:** A creator who prompts an AI, then substantially edits the output to add personal anecdotes, introduce sentence-length variation, and vary vocabulary will likely score in the `uncertain` band or even `likely_human`. The stylometric signal will detect the introduced variance; the LLM signal may still catch some structural patterns, but the combined score will be pulled toward the middle. This is probably acceptable behavior — the content is collaborative rather than purely AI-generated.

**Edge case 2 — Non-native English speakers writing formally:** A careful non-native English speaker whose writing is grammatically precise, uses formal vocabulary, and avoids colloquialisms will score AI-like on both signals. This is the most serious false-positive risk. The formal register triggers the LLM signal; consistent sentence structure and punctuation trigger the stylometric signal. This is exactly the scenario the appeal workflow exists for, and why the threshold for `likely_ai` is set conservatively at 0.70.

**Edge case 3 — Very short text (< 3 sentences):** The sentence-length variance metric requires multiple sentences to compute a meaningful variance. A haiku or single-sentence excerpt will produce unreliable stylometric scores. The system should acknowledge this limitation in the label for very short content.

**Edge case 4 — Poetry:** Poetry violates almost every assumption of the stylometric heuristics. It may have extreme sentence-length variance (intentionally), unusual punctuation, and high or low TTR depending on style. The LLM signal is also less reliable because it has fewer contextual cues. Poetry is a known weak spot.

---

## Rate Limiting

**Chosen limits:** 10 requests per minute, 100 requests per day per IP address.

**Reasoning:**
- A real human creator submits one piece of work at a time, perhaps a few times per day. 10/minute is generous even for testing.
- 100/day prevents a single IP from scanning large volumes of content to probe the detection system.
- The 10/minute burst limit targets automated submission scripts — a genuine human writer won't hit it.
- False positives from shared IPs (e.g., a university network) are acceptable because the limits are high enough that a single user on a shared IP won't exhaust the pool.

---

## AI Tool Plan

### M3 — Submission Endpoint + Signal 1

**Spec sections to provide:** Detection Signals (Signal 1 description), Architecture Diagram, API surface sketch.

**What to ask for:** Flask app skeleton with `POST /submit` route that validates `text` and `creator_id`, assigns a UUID `content_id`, calls a `classify_with_llm(text)` function, and returns structured JSON. Plus the `classify_with_llm` function itself that calls Groq with the structured prompt and parses `{"ai_probability": float}` from the response.

**How to verify:** Call the function directly in a Python REPL with 3 test inputs (clear AI, clear human, borderline). Inspect that the returned value is a float 0–1 before wiring into the route.

### M4 — Signal 2 + Confidence Scoring

**Spec sections to provide:** Detection Signals (Signal 2), Uncertainty Representation, Architecture Diagram.

**What to ask for:** `classify_with_stylometrics(text)` function computing sentence-length variance, TTR, and punctuation density into a single score; plus `compute_confidence(llm_score, stylo_score)` combining them 60/40 with threshold logic.

**What to check:** Run the four milestone test inputs through both signals separately, print each score, verify the combined score maps to the correct threshold band.

### M5 — Production Layer

**Spec sections to provide:** Transparency Label Design (all three variants), Appeals Workflow, Architecture Diagram.

**What to ask for:** `generate_label(attribution, confidence)` function mapping to the exact label text from the spec; `POST /appeal` endpoint with status update and log write; Flask-Limiter setup on `/submit`.

**How to verify:** Submit inputs that produce each of the three label variants and confirm the text matches the spec exactly. Submit an appeal and confirm `GET /log` shows the updated status and appeal reasoning.

---

## Stretch Features

*(Update this section before starting any stretch feature)*

No stretch features are planned in the initial implementation.
