# fitness-tracker

A personal health and fitness tracker — built April 2026.

## What's inside

A single-page HTML app with four tabs:

- **Schedule** — weekly workout plan with lifting, Pilates, walking, and Peloton rides
- **Exercises** — Workout A and B exercise details with sets, reps, and form cues
- **Nutrition** — interactive calorie deficit calculator with sliders for weight, goal, age, and deficit level
- **Meal Plan** — sample day of eating with Orgain protein powder incorporated into breakfast

## The plan at a glance

| Day | Activity |
|-----|----------|
| Monday | Walk to work + Lifting A (30 min) |
| Tuesday | Walk to work + Peloton Power Zone Endurance (30 min) |
| Wednesday | Pilates |
| Thursday | Walk to work + Lifting B (30 min) |
| Friday | Pilates + optional Peloton Power Zone (20 min) |
| Saturday | Rest |
| Sunday | Rest |

## How to use

Just open `index.html` in any browser — no build step, no dependencies, no server needed.

```bash
git clone https://github.com/YOUR_USERNAME/fitness-tracker.git
cd fitness-tracker
open index.html
```

## Updating the plan

All content lives in `index.html`. To update:
- **Schedule** — edit the `.day-row` blocks in the `#schedule` section
- **Exercises** — edit the `.ex-card` blocks in the `#exercises` section
- **Meal plan** — edit the `.meal-card` blocks in the `#meals` section
- **Nutrition calculator** — the `calc()` function in the `<script>` tag handles all the math

## Goals

- Starting weight: ~240 lbs
- Goal weight: ~200 lbs (targeting 30–50 lb loss)
- Age: 44
- Target: ~1 lb/week at moderate deficit (~500 cal/day)
- Daily protein target: ~160–170g
- Key supplement: Orgain Organic Protein powder (1 scoop/day in morning yogurt)
