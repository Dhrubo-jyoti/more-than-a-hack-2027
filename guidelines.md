# Guidelines — what Claude (or any AI tool) may and may not do on this project

Derived strictly from [RULES.md](RULES.md), [OVERVIEW.md](OVERVIEW.md), [DATA.md](DATA.md), and
[FAQ.md](FAQ.md). Where this document and those disagree, **the repo docs win** — update this
file, don't trust it blindly. Applies to Claude, ChatGPT, Gemini, Copilot, or any other AI tool
used anywhere in this project, by any team member.

---

## 1. The data firewall — never send competition match assets to a cloud AI

**Must never be pasted, uploaded, or otherwise sent to any third-party hosted/cloud LLM**
(Claude.ai, ChatGPT, Gemini, any hosted API, any SaaS tool with an AI feature) — in full or as an
extract:

- The tactical or broadcast **video** files.
- The **ATD** tracking files (`*_atd.xml`, `*_atd.txt`), raw or parsed, in full or in excerpted
  rows.
- The **Smart Tagging** XML, in full or in excerpted rows.
- The **homography** CSVs.
- Any **frame-level data derived from the above** — e.g. a CSV of ball positions, a plotted frame
  with tracked dots overlaid, a cropped screenshot of the video, a table of "frame X: ball at
  (a,b)".
- The **private ground truth** (not provided to us at all — moot, but stated for completeness)
  and anything that would reconstruct it.

This is a hard requirement from OVERVIEW.md, independent of everything else below: *"participants
shall not upload, disclose or otherwise make available to any third-party hosted or cloud-based
LLM any videos, datasets, confidential information or other restricted materials provided in
connection with the Competition, or any extracts thereof."*

**What this means practically, for Claude Code specifically:**
- Claude Code (this assistant) may read local files, run local scripts, and reason about code and
  numeric/statistical **summaries** it computes locally (counts, percentages, aggregate stats) —
  because those stay local, inside this tool-use session, and are not "uploaded to a third-party
  hosted LLM" the way pasting raw rows into a chat UI would be. Treat this as the dividing line:
  **local computation and summarization = fine; putting the raw asset content itself in front of
  a cloud model's eyes (verbatim rows, frames, video, pixels) = not fine.**
- Never pass a video frame, an image crop of the pitch, or a literal chunk of the ATD/XML/CSV
  content into a prompt to a *different* hosted AI tool (e.g. "let me ask GPT-4 to look at this
  frame") — that is exactly the disclosure the rule forbids.
- If a tool call would upload any of these files to an external service (an online converter, a
  cloud OCR tool, a third-party plotting-as-a-service site, a vision API) — don't. Keep all
  processing local or within this session's own sandboxed tool use.
- When in doubt whether a given summary is "an extract" of restricted material, treat it as
  restricted and keep it local only.

## 2. No hand-labelling the provided match — ever, by anyone, for any purpose

- Claude must never produce, suggest-as-final, or help a human produce a label for this match by
  looking at the video/frames and deciding what's "obviously" happening, and then having that
  judgement end up in the submission — directly or indirectly.
- This includes: looking at a frame or a short clip and saying "that looks like Barcelona has the
  ball here, just use H for frames 50000–50300" — even framed as "just a quick manual fix" or
  "just for validation." That is hand annotation regardless of who (human or AI) did the looking,
  per RULES.md: *"The labels must be produced by an algorithm... whether or not any answer key was
  involved."*
- No hand labels on this match may be used to **train or validate** anything either — not "just a
  handful to sanity check the model," not "just the obvious set pieces." Zero tolerance, any
  amount, any purpose.
- **What is fine:** Claude watching/inspecting data to *understand the format*, debug a loader, or
  help design a *general* rule or feature (e.g. "a throw-in typically follows a period of the ball
  being out of bounds near the touchline — here's a general rule for detecting that pattern
  algorithmically"). The line: understanding → OK. Turning a specific observation about *this
  match's specific frames* into a label, constant, or hardcoded rule tied to those frames → not
  OK.
- If Claude ever finds itself about to write something like `if frame in [50123, 50124, ...]:
  label = "H"` derived from "I looked and that's what it is" — stop. That is hand annotation
  wearing code's clothing.

## 3. Only the public 5-minute ground truth may be touched, and only as a sanity check

- `evaluation/gt/BAR-ATH_possession_gt_public.csv` (and its `_meta.json`) may be read and used to
  verify the pipeline runs correctly (right row count, right file format, labels broadly sane).
- It must **never** be used to tune a hyperparameter, pick between two model variants, train any
  component, or otherwise influence which answer gets submitted. The repo's own baselines prove
  that doing so picks the *wrong* answer on the private full match — this isn't just a rule, it's
  empirically the wrong move.
- Claude should flag it explicitly any time a task starts to look like "let's see which setting
  scores best on the public window" — that phrasing is the tell that a rule is about to be broken.
- The **private full-match ground truth does not exist in this environment** and never will. Any
  task that implies access to it (e.g. "what's the real accuracy on the full match") should be
  redirected to the public-window sanity check or bootstrap confidence intervals instead.

## 4. The submitted labels must come out of an algorithm, reproducibly

- Claude may write, improve, and run code that produces the possession file. It must not produce
  a final answer through any process a human (with or without AI) could not re-run and get the
  identical file from — "run this script against the provided data" must reproduce the submission
  exactly.
- No manual post-processing of an output file (hand-editing rows, "fixing" a few transitions by
  eye, smoothing by hand) is allowed, even as a one-off "just this once."
- Any external data Claude suggests using (pretrained models, other datasets, other matches'
  labels) must be about **other games**, never this one, and must be something the organisers
  could access and inspect — no private/internal-only models or datasets.

## 5. What Claude *can* do, unambiguously

To be clear this isn't a document about restricting Claude from being useful — most of the work is
fully in scope:

- Read, write, and refactor all code in this repository.
- Run local scripts, loaders, the evaluation/validation tools, and local EDA/plotting.
- Design algorithms, possession-detection heuristics, smoothing/segmentation approaches, and
  general-purpose rules (not tied to specific frames of this match).
- Use pretrained models, open-source libraries, or its own general football/CV/ML knowledge to
  propose approaches.
- Discuss, summarize, and compute statistics over the local data, as long as those summaries stay
  local to this session and are not relayed to another third-party AI tool/service.
- Help write the technical document, README, and reproducibility instructions.
- Be asked to double-check, against this file, whether a proposed step crosses a line — that's
  encouraged, not a nuisance.

## 6. If a request looks like it crosses a line

Claude should say so plainly and explain which section of this file (or of RULES.md/OVERVIEW.md)
it touches, rather than quietly complying or quietly refusing without explanation. "Open an issue"
is the organisers' own advice in RULES.md for anything genuinely ambiguous — when in doubt, flag
it rather than guess.
