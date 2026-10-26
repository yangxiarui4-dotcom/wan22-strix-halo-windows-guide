# Output grading standard and prompt guidelines (verified 2026-09/10)

Results of a controlled test battery on the gfx1151 / ROCm / Windows setup.
Every test used the same seed (42), 832×480, wan_2.1_vae, Enhance-A-Video 2.0,
euler; "draft" = lightx2v 4-step LoRA (steps 4, cfg 1.0), "final" = bare model
(steps 12, cfg 3.0, LoRA bypassed).

## Test battery verdicts

| # | Shot type | Frames | draft (4-step) | final (12-step) | Verdict |
|---|-----------|--------|----------------|-----------------|---------|
| T1 | Two gears meshing | 161 | broken physics | broken physics | NEVER generate — shoot real footage |
| T2 | Hands + tool (bolt tightening) | 81 | 1.5/2 | untested | composition OK, motion weak in draft |
| T3 | Single bearing + dolly-in, blurred bg | 81 | 2/2 | (expected better) | near-shippable even at draft |
| T4 | Engraved nameplate text | 33 | 1/2 | don't bet on it | words OK, digits wrong — overlay text in post |
| T5 | Coolant fluid over workpiece | 81 | 1.5/2 | — | great look, last ~20 frames loop — trim |

## Hard rules (production)

1. **Draft tier is a composition preview, never a deliverable.** The 4-step
   distilled LoRA has weak temporal consistency (micro-wobble on rigid outlines)
   and weak motion response (actions collapse into near-still poses).
2. **Final tier (12 steps, cfg 3.0, bare model) is the delivery floor.**
   Same prompt/seed at 12 steps was dramatically more stable and more "alive".
   Cost on this machine: ~2.5–3 h per 81-frame clip (cfg 3.0 doubles compute).
3. **Same seed does NOT transfer across step counts.** Changing 4→12 steps
   changes the composition entirely. "Approve draft, then render final" approves
   the *prompt*, not the exact framing.
4. **Never generate readable text or digits.** Word-level text may come out
   right; digits come out confidently wrong (engraved "2026" rendered as a
   corrupt glyph, stable across all frames). Add all text/nameplates/specs in
   post (ffmpeg drawtext / editor). Add "字幕/text" style negatives to prompts.
5. **Mechanical meshing (gears, mating parts) is beyond the model.** Teeth are
   malformed from frame 0 at any step count. Single rotating parts are fine.
6. **Keep clips ≤81 frames** (training length). At 161 frames the model falls
   into visible repetition in the back half. Even at 81, infinite-motion content
   (fluids) can loop in the last ~20 frames — trim in editing.
7. **Frame count = 4n+1** (33, 81). Plan shot lengths accordingly.

## Prompt template that survived testing (product hero shot)

```
industrial factory, close-up of a single steel ball bearing on a reflective
table, slow camera dolly-in, cinematic lighting, shallow depth of field,
blurred dark background
```

Why each clause earns its place:

- `single` — more than one moving rigid body = physics failures
- `slow camera dolly-in` — camera motion is well obeyed even at 4 steps
- `shallow depth of field` + `blurred dark background` — busy backgrounds
  wobble at draft tier; blurred bokeh hides residual jitter
- for actions: use visible-motion verbs ("actively turning", "pouring",
  "splashing"), not state verbs ("tightening" → static pose)

## Restart discipline

Changing LoRA state (bypass/restore) on a live ComfyUI process crashed the
interpreter during weight re-patching (ROCm native abort at weight-load 1%).
After changing LoRA/steps/cfg config, **restart the ComfyUI process** and run
the new config as the first job.
