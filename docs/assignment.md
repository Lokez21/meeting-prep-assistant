# Post Meeting Prep Assistant using Google AI Studio

## Overview

This assignment documents the design, development, and iteration of a **Post Meeting Prep Assistant** built using Google AI Studio, Gemini, and Stitch. The application accepts uploaded meeting documents (PDF notes and presentation slides) and returns structured post-meeting intelligence to help users prepare effectively for follow-up actions.

The project began with **Gemini 2.0 Flash** for rapid prototyping and prompt iteration, then migrated to **Gemini 2.5 Pro (pro-latest)** for the final version to benefit from deeper document reasoning and more accurate, thorough output generation.

---

## Architecture & Design Decisions

### System Architecture

```
User (Browser)
     │
     ▼
Upload Screen (Stitch UX)
     │  PDF notes + Slides + optional context
     ▼
Google AI Studio — Gemini 2.5 Pro (pro-latest)
     │  System prompt (Gem) enforces structured JSON output
     ▼
Structured JSON Response
     │  summary, risks, talking_points, next_steps, cover_image_prompt
     ▼
Results Screen (Stitch UX)
     │
     ▼
Export PDF / New Brief
```

### Model Selection

Model selection evolved across the project in two deliberate stages.

**Stage 1 — Gemini 2.0 Flash (prompt iteration phase)**

Flash was used during Iterations 1–3 for the following reasons:

- **Speed**: Flash returns structured outputs significantly faster than Pro, which matters when iterating on prompts and comparing outputs rapidly.
- **Cost**: Flash operates within the free tier quota (1,500 requests/day), making it safe for repeated experimentation without billing risk.
- **Structured output fidelity**: Flash reliably follows JSON schema constraints when given clear system instructions, which is sufficient for establishing baseline behaviour.

**Stage 2 — Gemini 2.5 Pro / `gemini-pro-latest` (final version)**

After completing prompt iterations, the model was switched to **Gemini 2.5 Pro (`gemini-pro-latest`)** for the following reasons:

- **Deeper document reasoning**: Pro performs more thorough analysis of complex or lengthy PDFs, surfacing non-obvious risks and connections that Flash may miss.
- **Accuracy**: Pro is less likely to produce surface-level or generic outputs on ambiguous documents — directly supporting the assignment's grounding requirement.
- **Thinking capability**: Gemini 2.5 Pro's extended thinking mode allows it to reason through document content before generating output, improving the quality of risk identification and next steps.
- **Trade-off acknowledged**: Pro is slower and consumes more quota than Flash. For a production system handling high request volumes, a hybrid routing strategy would make sense — Flash for short or simple documents, Pro for complex multi-document inputs above a token threshold.

### Grounding Strategy

A key design decision was the grounding policy — how strictly the model must adhere to the uploaded documents versus using general knowledge.

- **Iteration 1** took a strict "no inference" approach, which produced sparse and sometimes unhelpful outputs when documents lacked explicit detail.
- **Iterations 2 and 3** relaxed this to allow reasonable inference, with an `(inferred)` flag for any item not directly supported by the document. This balances factual grounding with practical usefulness.

### Output Schema Design

The schema was designed around the five fields required by the assignment brief:

| Field | Purpose | Design Note |
|---|---|---|
| `summary` | 2–3 sentence meeting overview | Constrained to prevent verbose overviews |
| `risks` | Specific, actionable risks | Must not be generic; severity implied by ordering |
| `talking_points` | Key points for follow-up discussion | Expandable — count scales to document complexity |
| `next_steps` | Concrete actions with owners | Time-bound where possible |
| `cover_image_prompt` | Image generation prompt for visual cover | Vivid, professional, meeting-relevant |

The decision to make item counts flexible (rather than fixed) came from observing that few-shot examples in the prompt anchored the model to a fixed number of items regardless of document richness. This was corrected in Iteration 3.

---

## Gem Design

A **Gem** (saved system instruction + model configuration) was created in Google AI Studio named `Meeting Prep Assistant`. The Gem encapsulates the full system prompt so it can be reused across sessions without re-entering instructions.

**Model config:**
- Model: `Gemini 2.0 Flash`
- Temperature: default (balanced)
- Output: structured JSON

---

## Prompt Iteration Log

### Iteration 1 — Baseline Schema

**Prompt:**
```
You are a Meeting Prep Assistant gem. When given a PDF of meeting notes and 
presentation slides, produce a structured JSON response output with the following fields:

{
  "summary": "2-3 sentence overview of the meeting context",
  "risks": ["risk 1", "risk 2", ...],
  "talking_points": ["point 1", "point 2", ...],
  "next_steps": ["step 1", "step 2", ...],
  "cover_image_prompt": "A descriptive prompt for generating a cover image"
}

Only use facts present in the uploaded documents. Do not infer or hallucinate.
```

**Reasoning:** Establish a baseline. Minimal instructions to observe raw model behaviour with strict grounding.

**Observed weakness:** The model anchored to the number of placeholder items shown in the example (3 risks, 5 points, 3 steps) and produced exactly that many items regardless of document content. Outputs felt mechanical. Cover image prompts were generic (e.g. "a professional meeting room").

---

### Iteration 2 — Grounding Rules + Quality Constraints

**What changed:**
- Added role context ("post-meeting intelligence")
- Relaxed grounding: allows general knowledge supplementary to document facts
- Added `(inferred)` flag for uncertain items
- Added qualitative rules: risks must be specific, next steps time-bound, cover prompt vivid
- Required `ONLY` JSON output (no preamble)

**Prompt:**
```
You are a Meeting Prep Assistant. Your job is to analyse uploaded meeting notes 
documents or slides (PDF notes, presentation slides, or any supporting materials) 
and generate structured post-meeting intelligence to help the user.

You may use both the uploaded documents and your general knowledge to enrich the 
output, but always prioritise facts from the uploaded documents. Never fabricate 
names, dates, figures, or decisions that are not present in the documents or 
reasonably inferable.

When the user uploads documents, respond ONLY with a valid JSON object matching 
this exact schema:

{
  "summary": "2-3 sentence overview of the meeting purpose and context",
  "risks": [
    "Risk 1 — brief description",
    "Risk 2 — brief description",
    "Risk 3 — brief description"
  ],
  "talking_points": [
    "Talking point 1",
    "Talking point 2",
    "Talking point 3",
    "Talking point 4",
    "Talking point 5"
  ],
  "next_steps": [
    "Action 1 — owner if known",
    "Action 2 — owner if known",
    "Action 3 — owner if known"
  ],
  "cover_image_prompt": "A detailed image generation prompt in one or two sentences"
}

Rules:
- If information is insufficient, use reasonable inference and flag with "(inferred)"
- Risks must be specific and actionable, not generic
- Next steps must be concrete and time-bound where possible
- Cover image prompt must be vivid, professional, and relevant to the meeting topic
```

**Improvement observed:** Richer, more actionable output. The `(inferred)` flag gave the model a sanctioned way to fill gaps without hallucinating. Cover image prompts became genuinely descriptive and meeting-specific.

---

### Iteration 3 — Flexible Item Counts (Final)

**What changed:** Decoupled item counts from example — explicitly stated that the example numbers are not targets. Added fallback instruction for when no document is uploaded.

**Full Prompt (production version):**
```
You are a Meeting Prep Assistant. Your job is to analyse uploaded meeting documents 
(PDF notes, presentation slides, or any supporting materials) and generate structured 
post-meeting intelligence to help the user prepare effectively.

You may use both the uploaded documents and your general knowledge to enrich the 
output, but always prioritise facts from the uploaded documents. Never fabricate 
names, dates, figures, or decisions that are not present in the documents or 
reasonably inferable.

When the user uploads documents, respond ONLY with a valid JSON object matching 
this schema:

{
  "summary": "2-3 sentence overview of the meeting purpose and context",
  "risks": [
    "Risk — brief description",
    "Risk — brief description"
  ],
  "talking_points": [
    "Talking point",
    "Talking point"
  ],
  "next_steps": [
    "Action — owner if known",
    "Action — owner if known"
  ],
  "cover_image_prompt": "A detailed image generation prompt in one or two sentences, 
  describing a professional visual that represents the theme and tone of this meeting"
}

Rules:
- The number of items in risks, talking_points, and next_steps should reflect the 
  complexity and content of the uploaded documents. Do not pad or truncate to match 
  any example count.
- If information is insufficient, use reasonable inference and flag it with 
  "(inferred)" at the end of that item.
- Risks must be specific and actionable, not generic.
- Next steps must be concrete and time-bound where possible.
- Cover image prompt must be vivid, professional, and relevant to the meeting topic.
- If no documents are uploaded, ask the user to upload meeting notes or slides 
  before proceeding.
```

**Improvement observed:** Model now scales section depth to document complexity. A one-page status update produces 2 risks and 2 next steps. A complex strategic document produces 6–7 risks. Output is proportionate rather than padded.

---

### Prompt Diff Summary

| Element | Iteration 1 | Iteration 2 | Iteration 3 |
|---|---|---|---|
| Role framing | Minimal | "Post-meeting intelligence" | Unchanged |
| Grounding | Strict (doc only) | Relaxed + `(inferred)` flag | Unchanged |
| Item counts | Implied by example | Still implied by example | Explicitly flexible |
| Quality rules | None | Specific, time-bound, vivid | Unchanged |
| No-doc fallback | None | None | Added |
| Output constraint | JSON only | JSON only (`ONLY`) | Unchanged |

---

## UX Design with Stitch

Three screens were designed and generated using **Stitch** based on structured natural language descriptions.

### Screen 1 — Upload / Input Screen

**Design rationale:** Two-column upload zone (PDF notes left, slides right) makes the dual-input requirement immediately clear. The optional context field allows users to add meeting title or attendee context without making it mandatory. The "Generate Meeting Brief" CTA is prominent and bottom-right aligned to follow natural reading flow.

*File: `screens/stitch-upload_input_screen.png`*

### Screen 2 — Processing / Loading Screen

**Design rationale:** Step-by-step progress list (Parsing PDF → Parsing slides → Identifying risks → Generating talking points → Preparing next steps) gives users visibility into what the model is doing rather than a blank spinner. This reduces perceived wait time and sets expectation for the output structure.

*File: `screens/stitch-processing_loading_screen.png`*

### Screen 3 — Results / Output Screen

**Design rationale:** The results screen mirrors the JSON schema in visual layout — summary at top, risks and talking points side-by-side (since they are complementary), next steps as a checklist (actionable), and cover image prompt at the bottom with a copy button. The "Export PDF" and "New Brief" actions are top-right, following standard document app conventions.

**Export PDF:** The "Export PDF" button allows users to download the full results page as a PDF file, giving them a portable, shareable record of the meeting brief without needing to copy content manually.

*File: `screens/stitch-results_output_screen.png`*

### Stitch Assessment

**Accelerated:** Stitch dramatically reduced the time needed to produce wireframe-quality screens with realistic placeholder content. The structured prompts translated directly to recognisable UI patterns without any CSS or HTML authoring.

**Constrained:** Stitch does not support interactive prototyping — the screens are static PNGs only. Iterating on fine layout details (e.g. adjusting column widths or reordering sections) requires re-generating from a modified prompt, which is slower than editing code directly. For a production project, Stitch would be used for initial layout validation before handing off to a component library (e.g. React + Tailwind).

---

## Reflection (≤ 800 words)

### How did your prompt evolve across iterations?

The prompt evolved along three dimensions: **grounding policy**, **output quality rules**, and **structural flexibility**.

Iteration 1 was deliberately minimal — a basic schema with a strict "no hallucination" rule. This gave a clean baseline but revealed an immediate problem: the model treated the example JSON as a specification, not an illustration. It produced exactly three risks and five talking points every time, regardless of what the document contained. The outputs were structurally correct but informationally thin.

Iteration 2 addressed the quality dimension. Adding role framing ("post-meeting intelligence to help the user prepare") gave the model a purpose beyond schema compliance. The `(inferred)` flag was the most impactful change — it gave the model a sanctioned mechanism for filling gaps rather than either hallucinating confidently or refusing to engage. The qualitative rules (specific risks, time-bound next steps, vivid cover prompts) improved output usefulness measurably.

Iteration 3 targeted the structural rigidity observed in both previous iterations. Explicitly stating "do not pad or truncate to match any example count" removed the few-shot anchoring effect. The model began producing proportionate outputs — a brief status update document yielded 2 risks; a detailed strategic planning document yielded 7. This is the behaviour a production system needs.

### What model selection trade-offs did you observe?

The project used two models deliberately. **Gemini 2.0 Flash** was used during the prompt iteration phase — its speed made it practical to run multiple prompt variants quickly and compare outputs side by side. Flash reliably followed the JSON schema and was sufficient for establishing baseline behaviour. However, on documents with nuance or ambiguity, Flash occasionally produced surface-level risks ("timeline may slip") without identifying the root cause embedded deeper in the document.

Switching to **Gemini 2.5 Pro (`gemini-pro-latest`)** for the final version produced noticeably more thorough outputs. Pro read further into documents, made more accurate inferences, and generated next steps that reflected the actual decision-making context rather than generic follow-ups. The extended thinking capability meant the model reasoned through the document before committing to output, which improved risk identification in particular.

The trade-off is speed and quota consumption. Pro is meaningfully slower and uses more of the free tier per request. For a production deployment, a routing strategy — Flash for documents under a token threshold, Pro for complex multi-document inputs — would balance cost against output quality.

### How did Stitch accelerate or constrain UX design?

Stitch's strength is speed-to-concept. Writing a structured English description of a screen and receiving a realistic-looking PNG in seconds is genuinely useful for communicating design intent early in a project. The screens produced for this assignment accurately conveyed the information hierarchy — upload inputs, processing state, structured results — without any frontend code.

The constraint is fidelity and iteration speed. Once a screen is generated, any change (reordering sections, adjusting spacing, renaming a button) requires re-prompting and re-generating. There is no direct manipulation. For a professional handoff, Stitch PNGs serve as reference material rather than implementation-ready designs. In a real project, these would feed into a Figma file or be translated directly to a component library.

### What would you change to productionise this system?

Several changes would be needed to move from this prototype to a production system:

**1. API layer:** Replace AI Studio's playground interface with direct Gemini API calls from a FastAPI backend. This enables request validation, error handling, retry logic, and response caching. The structured JSON output becomes a typed Pydantic model, enforced at the API boundary.

**2. File handling:** The current design assumes files are uploaded directly to the model context. In production, PDFs and slides would be pre-processed — text extracted, slides converted to images — before being passed to the model. This improves reliability on scanned or image-heavy documents.

**3. Authentication and rate limiting:** Multi-user deployment requires per-user API key management or a service account model, plus rate limiting to prevent abuse and control costs.

**4. Output validation:** The model's JSON output should be validated against a strict schema (e.g. using Pydantic or jsonschema) before being served to the frontend. If the model returns malformed JSON, the system should retry with a corrected prompt rather than surfacing an error to the user.

**5. Observability:** Logging each request's input documents, prompt version, model response, and latency would enable ongoing prompt quality monitoring and regression detection when the prompt is updated.

**6. Prompt versioning:** The three iterations documented here were managed manually. In production, prompts would be stored in version control with semantic versioning, and A/B testing would validate improvements quantitatively before promotion.

---

## Repository Structure

```
meeting-prep-assistant/
├── prompts/
│   ├── iteration_1.txt
│   ├── iteration_2.txt
│   └── iteration_3.txt
├── screens/
│   ├── stitch-upload_input_screen.png
│   ├── stitch-processing_loading_screen.png
│   └── stitch-results_output_screen.png
├── screenshots/
│   ├── AI-studio-version-history.png
│   ├── importing-stitch-screens-into-AI-studio-Gem.png
│   ├── prompt-iteration-1-default-design.png
│   ├── prompt-iteration-1-results-screen.png
│   ├── prompt-iteration-2-prepare-screen.png
│   ├── prompt-iteration-2-results-screen-1.png
│   ├── prompt-iteration-2-results-screen-2.png
│   ├── prompt-iteration-2-upload-screen.png
│   ├── prompt-iteration-3-results-screen.png
│   ├── raw-JSON-results-screen.png
│   └── raw-JSON-upload-screen.png
├── docs/
│   └── assignment.md
└── README.md
```

---

## Github repository
https://github.com/Lokez21/meeting-prep-assistant

---

## Cloud Run deployment

[https://post-meeting-prep-assistant-718013740090.europe-west1.run.app/](https://post-meeting-prep-assistant-718013740090.europe-west1.run.app/)

---

## Deliverables Checklist

- [x] `assignment.md` — this document
- [x] Stitch screens exported as PNG (3 screens)
- [x] AI Studio version history screenshots
- [x] Prompt iteration log (3 iterations with diffs)
- [x] GitHub repository link
- [x] Cloud Run URL
