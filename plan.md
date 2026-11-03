# Plan — Phase 1: Ball Possession

Companion to [explainer.md](explainer.md) (what/why, in plain language) and [task.md](task.md)
(the concrete checklist). This document is the *how we break the work down* layer.

**Working branch:** `eda-dhrubo`. Deadline: **November 17, 2026**.

## Guiding constraints that shape every phase below

- No hand-labelling of the provided match, ever, anywhere in the pipeline (not even "just to
  validate"). See RULES.md.
- The public 5-minute ground truth (`evaluation/gt/BAR-ATH_possession_gt_public.csv`) is for
  pipeline sanity-checks only — never for tuning or training. Any phase that touches it has to
  say explicitly "this is a sanity check" or it's a rules violation waiting to happen.
- Ranking metric is `spellF1` (possession-as-event), not `macroF1` (frame accuracy). Every design
  decision should be judged by "does this produce realistic possession spells", not "does this
  get more frames right".
- Target to beat: `spellF1 = 0.2433` (`baseline_smooth50`, full match).
- The evaluation script only ever needs the CSVs in `evaluation/`; it never needs the match data.
  That means EDA/modelling work and scoring work can proceed independently once we have *a* CSV.

## Phase 0 — Environment & orientation (today)

Get the repo, data, and tooling in a state where we can run the existing baseline and scorer
end-to-end before writing anything new. This is a trust-building step: if we can't reproduce the
numbers the README quotes, nothing downstream is worth doing yet.

Outputs: `pip install` done, `data/` confirmed populated and matching the structure in DATA.md,
baseline + scorer run successfully, numbers match the README's published figures (on the public
window).

## Phase 1 — Exploratory Data Analysis (this branch)

The goal of EDA here is narrow and specific: build an intuitive, verified picture of *the four
provided assets* and of *what the public ground truth actually looks like*, so that every later
modelling decision is informed rather than guessed. We are explicitly **not** trying to reverse-
engineer the private ground truth or hand-tune to the public 5 minutes — we're trying to
understand the inputs.

Sub-areas, roughly independent of each other:

1. **Tracking (ATD) shape and quality.** How many track slots are actually occupied over time,
   how often the ball is missing and in what patterns (random blips vs. long gapsclustered around
   certain match moments), what a "team in possession" signal built from nearest-player-to-ball
   looks like raw, and how noisy/flickery it is frame to frame.
2. **Smart Tagging structure.** Which codes exist, their rough frequency and duration, how often
   `Team` is `N/A` (and when — the DATA.md flag that *every* throw-in is untagged for team is a
   concrete thing to confirm and then plan around), and how tagging intervals relate in time to
   candidate possession changes from (1).
3. **Homography sanity.** Confirm `valid`/`reprojection_error` behave as documented (tactical
   100% valid, broadcast ~66%), and get one or two frames projected onto video stills as a visual
   gut-check that coordinates mean what we think they mean.
4. **Public ground truth, as a worked example.** Look at the 17 possessions in the first 5
   minutes: how they start/end relative to Smart Tagging intervals and tracking signals, how the
   "nearest-player-to-ball" signal behaves right at a genuine hand-over vs. in the middle of a
   spell, what a `D` stretch looks like in the raw tracking. This is the one piece of ground truth
   we're allowed to look at directly — use it to calibrate intuition, not parameters.
5. **Baseline failure modes.** Run the four shipped baselines, look at *where* they disagree with
   the public ground truth (not just the aggregate score) — which kinds of frames/transitions they
   get wrong, to find the next lever to pull.

Output of Phase 1: a short written summary (can live in a notebook or a `docs/eda-notes.md`) of
findings per sub-area above, plus a prioritised list of "signals worth building on" that feeds
Phase 2. No modelling code is required to be novel yet — reusing/extending `starter/*` loaders is
expected and intended.

## Phase 2 — Approach design

Translate the Phase 1 findings into one or more concrete modelling approaches, reasoned about
against the metric (not against per-frame accuracy). This is a short design phase, not
open-ended exploration — the point is to pick a direction before sinking time into it.

Decisions to make explicitly here, each with a one-line rationale tied back to an EDA finding:

- How to produce a per-frame team signal when the ball is visible (nearest player is the given
  baseline — is there a cheap improvement, e.g. weighting by recent ball velocity/direction,
  using both teams' convex hulls, etc.?).
- How to fill/handle the ~41% of frames where the ball is invisible (hold last known team? model
  possession as a state machine with inertia? use Smart Tagging as a prior during gaps?).
- How to decide `D` vs. `H`/`A` — i.e., a model of "ball alive vs. dead" independent of which team
  has it, since the baseline conflates "ball not visible" with "dead ball" and that's a known
  weakness (see `starter/baseline.py` docstring: "no notion of a dead ball beyond the tracking
  losing the ball").
- How to turn a noisy frame-level signal into clean possession spells (bigger/adaptive smoothing
  window vs. an explicit segmentation/state model vs. change-point detection) — this is the one
  decision Phase 1's baseline-disagreement analysis should drive most directly, since it's
  exactly what separates `spellF1 = 0.05` from `spellF1 = 0.24` on the *same* frame-level signal.
- Where (if anywhere) Smart Tagging and homography earn their keep vs. add complexity for little
  gain.

Output: a short written approach doc (e.g. `docs/approach.md`) naming the chosen direction(s), and
an explicit list of what's deliberately deferred ("dropped", in SUBMISSION.md's language — this
maps directly into the technical document we eventually submit).

## Phase 3 — Implementation

Build the chosen approach as a script/pipeline that:
- reads only the provided assets (plus, optionally, outside data about *other* matches),
- produces a `frame,label` CSV for the full 199,319 frames,
- is fully regenerable from a single documented command (a rules requirement, not just hygiene).

Iterate in small, independently testable increments (e.g.: get a raw per-frame signal working and
validated → add dead-ball detection → add smoothing/segmentation → add Smart-Tagging-informed
refinements), scoring against the public window after each increment purely as a format/sanity
check, never as a selection criterion between variants.

## Phase 4 — Validation, robustness, and anti-overfitting checks

- Run `validate_submission.py` on every candidate file — mechanical, required, cheap.
- Sanity-check reported possession counts/durations against footballing intuition (not against
  the public ground truth's own counts, which are intentionally unpublished) — e.g. "does a
  90-minute half really have ~900 possessions, or does that smell like under-smoothing?"
- Use `--bootstrap` on the public window to see whether an apparent improvement is inside the
  noise band before trusting it, per EVALUATION.md's explicit warning that public-window
  differences below ~0.04 spellF1 are noise.
- Explicitly re-check against the Phase 0 constraint list: no hand labels anywhere, no tuning
  against the public window, reproducible from the repo.

## Phase 5 — Deliverables packaging

- Finalise the submission CSV, named after the team.
- Clean up the repository so "clone → install → one command → same CSV" genuinely works for
  someone else.
- Write the short technical document (approach, assets used/dropped, how noise was handled,
  possession rules implemented, limitations, reproduction steps) — this should mostly be a
  distillation of the Phase 2 approach doc plus what actually happened in Phase 3/4.
- Submit via the form before **November 17, 2026**.

## Open decisions / risks to revisit

- Team composition requirement (3–5 members for Innovation Challenge/Final Pitch eligibility) is
  outside this plan's scope but worth confirming isn't blocking.
- Whether to invest in the broadcast video/tracking at all — DATA.md flags it as unsynced to
  tactical with no provided mapping; worth an explicit go/no-go decision in Phase 2 rather than
  drifting into it.
- Compute/tooling: notebooks want `kloppy` + `matplotlib` (`requirements-starter.txt`); core
  scoring pipeline only needs `numpy`/`pandas`. Keep the two concerns separable so the
  regenerate-submission command stays light.
