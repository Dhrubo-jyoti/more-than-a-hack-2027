# Explainer — what this project actually is (plain language)

## 1. The one-sentence version

Watch a football match frame by frame and say, for every single frame, "Barcelona has the ball",
"Athletic has the ball", or "the ball is not in play right now". Do that for all 199,319 frames,
put it in a CSV, and a script scores how close you got to the truth.

## 2. The match

- FC Barcelona (**home**, label `H`) vs Athletic Club (**away**, label `A`).
- LaLiga 2024-25, matchday 2.
- Filmed by two cameras: a **tactical** (wide, fixed, overhead-ish) camera and a **broadcast**
  (normal TV) camera.
- The tactical video is the one that matters. Every row we submit = one frame of that video,
  in order, frame 0 to frame 199,318. The broadcast video is a bonus extra; it has a different
  number of frames and is not lined up with the tactical one at all, so it's harder to use.

## 3. What we are handed vs. what we have to figure out

We are **given**, already computed by Metrica's automatic systems (nobody manually reviewed any
of this, so it has mistakes):

- **Tracking (ATD):** for every frame, the (x, y) position and speed of the ball and of ~22
  players, plus which of the two teams each player belongs to. No player names — just anonymous
  "track slots" that can jump, disappear, or swap identity.
- **Ball position** is part of that tracking, but **the ball is simply missing on about 41% of
  the frames that get scored** — it's small, fast, and often hidden behind players. That's the
  single hardest fact about this whole project.
- **Smart Tagging:** 638 automatically-detected "phases" like "Build Up", "Set Pieces",
  "Attacking Transition", each with a team tag, a time range, and some extra labels. Think of it
  as a rough, sometimes-wrong summary of what tactical thing was happening and who was doing it.
- **Homography:** for every frame, a little matrix that converts a pitch position (0 to 1 on each
  axis) into a pixel position in the video. Lets you draw dots on top of the video to sanity-check
  yourself, or go the other way (pixel → pitch).

We have to **figure out**:

- Who has the ball, frame by frame, using only the above (or our own extra processing of the
  video, if we want to go further).
- When nobody "controls" the ball (it's in the air, bouncing around, just been touched but not
  controlled) — those frames still need one of the 4 labels, decided by rules we design.
- How to turn noisy, flickery, frame-by-frame guesses into clean, realistic chunks of
  "possession" rather than a label that changes 50 times a second.

## 4. The exact output we must produce

A CSV, header `frame,label`, exactly 199,319 data rows (frame 0 through 199,318), label is one of:

| label | means |
|---|---|
| `H` | Barcelona has the ball, it's in play |
| `A` | Athletic has the ball, it's in play |
| `D` | dead ball — throw-in about to happen, ball out, half-time moment, etc. |
| `X` | "I don't know" — we refuse to guess on this frame |

Only ~159k of the 199k frames are actually scored (roughly: during the two halves, not warm-up /
half-time / after full-time). On the frames that aren't scored, writing `X` costs nothing, so we
should always write `X` there rather than guess.

## 5. Why `X` is not "giving up" — it's a real tool

- Under the *secondary* metric (`macroF1`), an `X` on a scored frame counts as simply wrong —
  same as guessing badly.
- Under the *main* metric (`spellF1`), an `X` is invisible — the scorer secretly replaces it with
  whatever label we gave right before it (in that half of the match).
  - So: saying `X` in the **middle** of a spell of one team having the ball is basically free —
    the scorer will just extend the previous label through it, which is probably correct anyway.
  - Saying `X` exactly **at the moment possession changes hands** is bad — the scorer will wrongly
    extend the *old* team's possession through the handover.
  - **Rule of thumb:** go quiet (`X`) when unsure *and nothing seems to be changing*. Force a
    guess when unsure *and it looks like a changeover is happening*.

## 6. How we're graded (and why it's not "accuracy")

Two numbers, but only one decides the ranking:

- **`spellF1` (the one that matters for ranking).** Possession, to a coach, is "Barcelona had the
  ball for 14 seconds, then Athletic had it for 6 seconds" — a handful of *events* across the
  match, not hundreds of thousands of frames. `spellF1` treats the match as a list of these
  possession-events ("spells") and checks how many of our spells line up with the real ones in
  time. Get the right number of real, well-timed possessions, and this score is high. Spit out a
  flickery mess of hundreds of tiny fake possessions, and this score tanks — **even if almost every
  individual frame was labelled correctly.**
- **`macroF1` (secondary, only breaks ties).** The boring, frame-by-frame "did you get this exact
  frame right" score, averaged fairly across H / A / D (so that the fact that "dead ball" is a
  less common state doesn't make the score easy to game).

The repo proves with its own example that these two scores can *disagree on which answer is
best* — the same simple algorithm, run with 4 different amounts of "smoothing" (how much we blur
flickery frame guesses into steady chunks), gets ranked in a completely different order by each
metric. **This is the single most important thing to understand before writing any code**: our
system needs to think in terms of possessions-as-events, not frames-as-independent-guesses.

## 7. The floor we need to beat

A dead-simple baseline already ships in this repo: "whichever player is nearest the ball, that
team has it" + "smooth that over some number of frames so it doesn't flicker". With the best
smoothing amount tried, it scores `spellF1 = 0.2433` on the full match. **That's the number to
beat.** Nothing fancy — distance-to-ball plus majority vote in a window.

## 8. Rules, in plain speech

- Any tool, model, language, or external data is fair game — **as long as it's about other
  matches**, and as long as it's something the organisers could, in principle, look at and run.
- **Zero hand-labelling of this match**, period. Not for the final answer, not as training data,
  not as a "just to validate" set. If a human looked at this match's video and typed down an
  answer anywhere in the pipeline, that's disqualifying — even if the human was "just checking".
  (Watching the video to understand it / debug code is fine. Writing down what you saw as a label
  or a frame-specific rule is not.)
- We only get real answers ("ground truth") for the **first 5 minutes** of the match. That's for
  sanity-checking our pipeline (right frame count, right file format, labels doing roughly the
  right thing) — **never** for tuning parameters or training a model. The repo proves that tuning
  against those 5 minutes literally picks the *wrong* answer compared to what wins on the full,
  private match.
- Whatever we submit, our own repo must be able to regenerate that *exact* file by running some
  command. No hand-massaging the output afterward.
- We must end up with: (1) the CSV, (2) a public code repo, (3) a short write-up of what we did
  and why.

## 9. What "good" looks like, intuitively

Not "label every frame correctly" — more like "correctly spot every time the ball changed hands,
roughly at the right moment, without inventing possessions that didn't happen." A system that
nails frame accuracy but imagines 900 fake tiny handovers will score *worse* than a cruder system
that gets the big picture of "who had it, from when to when" right. Smoothing / segmenting /
state-modelling the raw frame-by-frame signal into clean possession-chunks is not an optional
nice-to-have — it is most of the actual problem.
