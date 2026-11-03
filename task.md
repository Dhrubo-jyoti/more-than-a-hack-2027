# Tasks

Companion to [explainer.md](explainer.md) (what/why) and [plan.md](plan.md) (how we break it
down). This is the actionable checklist, kept in sync with the plan's phases. Check items off as
they're done; add new ones as Phase 1 findings suggest new work — don't let this file go stale.

**Branch:** `eda-dhrubo`.

---

## Phase 0 — Environment & orientation

- [ ] Confirm Python version (`python --version`) is 3.8+, ideally 3.9+ for notebook tooling.
- [ ] `pip install -r requirements.txt` (core: numpy, pandas).
- [ ] `pip install -r requirements-starter.txt` (adds kloppy, matplotlib for the notebooks).
- [ ] Confirm `data/` matches the structure in DATA.md (`tactical/` and `broadcast/` subfolders,
      correct filenames) — already looks populated; just verify file sizes/names are sane.
- [ ] Run `python evaluation/run_evaluation.py` on a clean checkout of the four shipped baselines
      and confirm the printed board matches the numbers in README.md / EVALUATION.md (public
      window: `baseline_smooth125` best at spellF1 0.4242, etc.).
- [ ] Run `python starter/baseline.py` to regenerate the four baselines locally from the actual
      match data, confirm it reproduces the same files/scores.
- [ ] Run `python evaluation/validate_submission.py evaluation/results/baseline_smooth50.csv` and
      read its output end to end — this is the exact check our own future submissions must pass.
- [ ] Skim `evaluation/possession_metrics.py` once (the metric definitions are the actual spec —
      EVALUATION.md says so explicitly) — no need to understand every line yet, just know it
      exists and is authoritative over the docs if they ever disagree.

## Phase 1 — EDA

### 1a. Tracking (ATD)

- [ ] Run `python starter/load_atd.py` — confirm the alignment check passes
      (`video_frame = frame_id - 1`) and read the printed landmark frames.
- [ ] Load the tactical ATD (via kloppy, `starter/load_atd.py:load()`, or the raw-text approach in
      `starter/baseline.py`) and compute, per frame: whether the ball is visible, and how many
      track slots are occupied.
- [ ] Plot/tabulate ball-visibility over time across the match — is missingness random, or
      clustered (e.g. around certain phases, crowded-box moments, aerial play)? Compare the two
      halves.
- [ ] Reproduce the "ball visible on 56.9% of all frames / 59.3% of graded frames" figures from
      DATA.md on our own load, as a correctness check on our understanding of the file format.
- [ ] Build the raw "nearest player to ball → team" signal (reuse
      `starter/baseline.py:nearest_player_labels`) and look at it unsmoothed: how often does it
      flip team within a short window (flicker rate), and does that match the "the raw baseline
      reports way more possessions than the match really has" claim in FAQ.md?
- [ ] Count occupied track slots per frame; confirm the documented "~22 during play, mean 20.5
      over graded frames, occasionally >22" and see how it dips at halftime — useful as an
      independent "is the game being played right now" signal.

### 1b. Smart Tagging

- [ ] Run `python starter/load_smart_tagging.py` and confirm the time-base check passes
      (`round(seconds * 25) == video frame`, verified against both kickoffs).
- [ ] Load all 638 Smart Tagging instances; tabulate counts per `code` and confirm they match the
      table in DATA.md (sanity check on parsing).
- [ ] Look specifically at `SET PIECES` instances: confirm the `Type` breakdown (throw-ins,
      corners, etc.) and confirm the DATA.md claim that **every throw-in is `Team = N/A`** —
      since throw-ins are exactly the dead-ball→live-ball handovers we most need to label well.
- [ ] For a handful of non-`SET PIECES` phases (`BUILD UP`, `ATTACKING TRANSITION`, etc.), check
      how their time interval relates to the raw nearest-player-to-ball signal from 1a — does the
      tagged team mostly agree with the signal during that interval, or do phases span changes of
      possession?

### 1c. Homography

- [ ] Run/inspect `starter/load_homography.py`; confirm tactical `valid` is 1 on effectively every
      frame, and (if we end up loading broadcast too) confirm its ~65.8% figure.
- [ ] Pick 2–3 interesting frames (e.g. a clear throw-in, a clear open-play moment) and project
      the ball + nearest player positions onto the video frame as an image, purely as a visual
      gut-check that pitch coordinates and frame indices mean what we think.

### 1d. Public ground truth as a worked example

- [ ] Load `evaluation/gt/BAR-ATH_possession_gt_public.csv` and its `_public_meta.json`; confirm
      the published figures (17 possessions, mean 14.2 s, H 3454 / A 2577 / D 1469 over frames
      11185–18684).
- [ ] For each of the 17 possessions, note how the raw nearest-player signal (1a) and any
      overlapping Smart Tagging phase (1b) behave right at the transition into and out of it —
      goal: a qualitative sense of "what a real hand-over looks like in the raw signals", not a
      tuned threshold.
- [ ] Look at at least one `D` stretch inside the public window and describe what the raw tracking
      is doing during it (ball missing? ball visible but no clear nearest player? a Smart Tagging
      `SET PIECES` tag present?).
- [ ] Explicitly note: nothing from this sub-task may turn into a hand-picked threshold, constant,
      or rule that is specific to *this* match's public window — findings here inform *general*
      design choices (Phase 2), not match-specific tuning. Flag anything that feels like it's
      drifting into that territory.

### 1e. Baseline failure modes

- [ ] Score each of the four shipped baselines against the public ground truth with
      `run_evaluation.py --bootstrap 1000` and note the confidence intervals, not just the point
      estimate.
- [ ] For `baseline_raw` vs `baseline_smooth50`, diff their per-frame labels inside the public
      window and characterise *where* they disagree (near transitions? in the middle of long
      spells? during `D` stretches?).
- [ ] Write up: which single change (dead-ball detection vs. smarter smoothing vs. Smart-Tagging
      prior vs. something else) looks like the highest-leverage next step, with the EDA evidence
      that supports it.

### 1f. Wrap-up

- [ ] Write a short EDA notes doc (`docs/eda-notes.md` or similar) summarising findings from
      1a–1e and the prioritised list of "signals worth building on".
- [ ] Review the notes against the "no hand labels on this match, no tuning on the public window"
      constraints one more time before treating any of it as a design input for Phase 2.
- [ ] Commit the EDA work (notebooks/scripts/notes) on `eda-dhrubo`.

---

## Later phases (tracked here, detailed in plan.md)

- [ ] Phase 2 — write `docs/approach.md` with the chosen modelling direction(s).
- [ ] Phase 3 — implement the pipeline, regenerable from one command.
- [ ] Phase 4 — validate (`validate_submission.py`, bootstrap checks, anti-overfitting re-check).
- [ ] Phase 5 — package deliverables (CSV, public repo, technical document) and submit before
      **November 17, 2026**.
