# Post Meeting Prep Assistant

A post-meeting intelligence tool built with **Google AI Studio**, **Gemini**, and **Stitch**. Upload meeting notes and presentation slides (PDF) and receive a structured JSON brief — summary, risks, talking points, next steps, and a cover image prompt.

---

## How it works

1. Open the Gem in Google AI Studio (`Meeting Prep Assistant`)
2. Upload your PDF meeting notes and/or slides
3. Receive a structured JSON response ready for review or export

```text
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

---

## Output schema

```json
{
  "summary": "2-3 sentence overview of the meeting purpose and context",
  "risks": ["Risk — brief description"],
  "talking_points": ["Talking point"],
  "next_steps": ["Action — owner if known"],
  "cover_image_prompt": "Vivid image generation prompt for a visual cover"
}
```

Item counts in `risks`, `talking_points`, and `next_steps` scale to the complexity of the uploaded documents.

---

## Models used

| Phase                   | Model                                    | Reason                                                     |
| ----------------------- | ---------------------------------------- | ---------------------------------------------------------- |
| Prompt iteration (1–3)  | Gemini 2.0 Flash                         | Speed and free-tier quota for rapid experimentation        |
| Final version           | Gemini 2.5 Pro (`gemini-pro-latest`)     | Deeper document reasoning and extended thinking            |

---

## Prompt iterations

Three prompt versions are in [`prompts/`](prompts/), each building on the last:

| File                                                          | Change                                            |
| ------------------------------------------------------------- | ------------------------------------------------- |
| [`iteration_1.txt`](prompts/iteration_1.txt)                  | Baseline schema, strict grounding                 |
| [`iteration_2.txt`](prompts/iteration_2.txt)                  | Role framing, `(inferred)` flag, quality rules    |
| [`iteration_3.txt`](prompts/iteration_3.txt)                  | Flexible item counts, no-document fallback        |

---

## UX screens (Stitch)

Designed in Stitch from structured English descriptions — static PNG wireframes.

| Screen              | File                                                                                                          |
| ------------------- | ------------------------------------------------------------------------------------------------------------- |
| Upload / Input      | [`screens/stitch-upload_input_screen.png`](screens/stitch-upload_input_screen.png)                            |
| Processing / Loading| [`screens/stitch-processing_loading_screen.png`](screens/stitch-processing_loading_screen.png)                |
| Results / Output    | [`screens/stitch-results_output_screen.png`](screens/stitch-results_output_screen.png)                        |

---

## Repository structure

```text
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

## Full documentation

See [`docs/assignment.md`](docs/assignment.md) for the complete write-up: architecture decisions, grounding strategy, prompt iteration log, Stitch assessment, and production roadmap.
