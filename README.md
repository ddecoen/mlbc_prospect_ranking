# MLBC Prospect Ranker

A browser-based prospect ranking engine for the **Minor League Baseball Club (MLBC)** simulation game. Upload your league export files and get instant, data-driven prospect rankings — no server, no Python, no installs required.

🔗 **Live app:** [https://ddecoen.github.io/mlbc_prospect_ranking](https://ddecoen.github.io/mlbc_prospect_ranking)

---

## How to Use

1. Export two CSV files from your MLBC sim:
   - `League_Roster_XXXX.csv` — full league player roster with all edits/grades
   - `Pro_Years_Report_XXXX.csv` — pro years report (filters to prospects with 0 pro years)
2. Open the app and upload both files
3. Rankings generate instantly in your browser — filter, sort, and export to CSV

---

## Model Philosophy

The model is built around one core principle: **grades are grades**. A player's edits represent their true ability ceiling regardless of what level they play at. Level only tells you *when* you'll get the value, not *how much* value there is. No level-based adjustments are applied to scores.

The model separates hitters and pitchers completely, scoring each against the full non-ML prospect pool.

---

## Hitter Model

### HIT Score (90% of overall)

The offensive grade is built from five components on a **20–75 scale**, calibrated to the full non-ML prospect population:

| Component | Weight | Stat | Logic |
|-----------|--------|------|-------|
| OBP Grade | 35% | `(OBP_vL × 0.25) + (OBP_vR × 0.75)` | On-base is the most predictive offensive rate stat |
| XBH Grade | 33% | `HR×4 + 3B×3 + 2B×2` | Extra base value mirrors the sim's own fantasy formula |
| K/BB Grade | 12% | `B_SO / B_BB` | Plate discipline — lower is better |
| GB% Grade | 12% | `B_GB` | Lower GB% = fly ball tendency = more power output |
| Pull Grade | 8% | `Pull%` | Pull hitters generate more HR; sim rewards pull power |

**Hit Tool Penalty:** Players with `B_H < 170` receive a contact risk penalty (−1 to −4 points) applied to the final overall score. Below 155 is a significant red flag; below 140 is severe.

### FLD Score (10% of overall)

Defense is real but worth approximately 10% of a prospect's value. The sim's fantasy scoring is entirely offensive, but good defense helps pitchers and prevents runs.

Each position uses **position-appropriate defensive weights** with range z-scored within position group (so a 1B's range is compared to other 1Bs, not shortstops):

| Position | Primary Weight | Secondary |
|----------|---------------|-----------|
| C | Arm 65% | Hands 25% |
| SS | Range 40% | Arm 25%, Run 20% |
| 2B | Range 40% | Arm 25% |
| 3B | Arm 45% | Range 30% |
| CF | Range 40% | Run 35% |
| RF | Arm 45% | Range 30% |
| LF | Range 40% | Run 30% |
| 1B | Hands 55% | Range 25% |

**Throw hand constraint:** Left-handed throwers cannot play meaningful infield positions other than 1B. LH throwers coded at 2B/3B/SS use OF range as their range component, which correctly penalizes the positional mismatch.

### Bonuses (additive to overall)

These bonuses recognize archetypes the sim rewards but raw grades don't fully capture:

| Bonus | Trigger | Points | Rationale |
|-------|---------|--------|-----------|
| **Positional Value** | C, SS, CF, 2B, 3B, RF, LF, 1B | +3.0 to −0.5 | Scarce defensive positions are worth more at equivalent offensive output |
| **True Power** | OPS ≥ 1.050 | +3.0 | Generational bat — top 0.3% of prospect pool |
| | OPS ≥ 1.000 | +1.5 | Elite power prospect |
| | OPS ≥ 0.950 | +0.5 | Plus power |
| **True Leadoff** | OBP ≥ 0.390 + Run ≥ 5.5 | +2.0 | Both OBP and speed elite simultaneously |
| | OBP ≥ 0.380 + Run ≥ 5.0 | +1.0 | Legitimate leadoff profile |
| **Bat Hand** | Switch | +1.0 | Platoon advantage every at-bat |
| | Left-handed | +0.5 | Slight platoon advantage vs. majority RHP |

**Positional value premiums:**
- C: +3.0 — catching defense directly impacts pitcher ERA; hardest position to find offense
- SS: +2.0 — most difficult defensive position; offensive SS are transformative
- CF: +1.5 — covers the most ground; leadoff/speed value compounds
- 2B/3B: +0.5 — premium over corner positions
- RF/LF: 0 — pure bat positions
- 1B: −0.5 — easiest defensive position; bat must carry the value

---

## Pitcher Model

### STUFF Score

The pitcher model is built on one core insight: **in a sim with fixed pitch probability distributions, W_OPS is a complete and sufficient quality metric.**

The sim engine selects pitches randomly according to each pitcher's usage percentages (`P1_Qual` through `P6_Qual`). You cannot instruct your pitcher to throw his fastball more often — the probabilities are fixed. This means:

- The **best pitch** is already weighted into W_OPS by how often the sim calls it
- The **worst pitch** is already weighted into W_OPS by how often the sim calls it
- **Pitch count** doesn't matter — a 3-pitch pitcher who dominates with all three is equal to a 5-pitch pitcher with the same W_OPS
- **Top-2 or bottom-1 metrics** double-count what W_OPS already captures

The only independent signal is **K/BB** — walks and strikeouts are not captured in per-pitch OPS splits, and command is genuinely pitcher-controlled regardless of what the sim randomly selects.

| Component | Weight | Stat | Logic |
|-----------|--------|------|-------|
| W_OPS | 85% | Usage-weighted OPS against across all pitches | Complete pitch quality signal — captures best, worst, and mix simultaneously |
| K/BB | 15% | `P_SO / P_BB` | Only independent signal; command is pitcher-controlled |

**How W_OPS is computed:**

W_OPS uses the actual batter handedness distribution from the league, not a naive 50/50 split:

- **LHP** faces 76.4% RHH (R batters + switch hitters batting right vs LHP)
- **RHP** faces 64.7% RHH

For each pitch: `OPS_weighted = OPS_vL × handedness_weight_L + OPS_vR × handedness_weight_R`

Then: `W_OPS = Σ(pitch_usage × OPS_weighted)` across all pitches

This gives the true expected OPS on any randomly selected pitch, accounting for the actual distribution of batter types the pitcher will face.

**Why not GB%?** Ground ball rate is already implicit in W_OPS — a ground ball specialist allows fewer hits and fewer home runs, which shows up directly in his pitch OPS splits. Adding GB% as a separate component double-counts the same information while unfairly disadvantaging strikeout pitchers who achieve the same W_OPS through a different approach.

**Why not K rate separately?** Strikeouts are the best possible outcome (no baserunner, no defense required), but their value is already reflected in the per-pitch OPS against. A pitcher who strikes out 200 batters per 600 has a lower OPS against than one who strikes out 80, all else equal.

**Endurance (END) as a gate only:**
- END ≥ 5.0 → Starter-eligible, no adjustment
- END < 5.0 → Reliever penalty (−5 points)

Endurance affects *volume* of contribution, not *quality*. The gate prevents relievers from being overrated relative to starters but does not otherwise favor workhorse arms. A pitcher with END=5.5 and elite W_OPS is correctly ranked above one with END=8.0 and mediocre W_OPS.

---

## Grade Scale

All grades use a **20–75 scale** (matching traditional scouting 20-80 grades minus the very top tier, which is reserved for generational MLB players rather than prospects):

| Grade | Meaning |
|-------|---------|
| 80 | Reserved for active ML superstars in the all-time tier |
| 75 | Elite — top 1–3% of prospect pool |
| 70 | Plus-plus — top 5–8% |
| 65 | Plus — top 10–15% |
| 60 | Above average — top 25% |
| 55 | Solid average |
| 50 | Average |
| 45 | Below average |
| 40 | Fringe |
| 30 | Poor |
| 20 | Well below average |

Bins are calibrated to the **full non-ML prospect population** so grades are consistent year over year regardless of which season's data is uploaded.

---

## Why These Decisions

**No level adjustment:** The MLBC edits represent a player's true ability grades — they do not change as a player moves from A-ball to AAA. A 19-year-old in A-ball with elite grades should not be ranked below a 24-year-old in AAA with average grades just because he hasn't been promoted yet. Level tells you *when* you get the value. The ranking tells you *how much* value there is.

**90/10 bat/glove for hitters:** Analysis of the sim's fantasy scoring formula (1B=1, 2B=2, 3B=3, HR=4, BB=1, SO=−1) shows that offensive production drives virtually all measurable value. The run value difference between elite and poor defense is approximately 3–4 runs per season, while a single extra OPS point represents ~0.4 runs per 600 AB. The bat-to-glove run value ratio is approximately 10:1.

**Positional premiums instead of defensive scoring inflation:** Rather than letting fielding grades inflate overall scores for corner players (a known bug in earlier versions of this model), positional scarcity is handled as a clean additive bonus. This correctly ranks a .850 OPS shortstop above a .850 OPS first baseman without letting IF_Rng artificially boost a first baseman's score.

**W_OPS as the complete pitcher signal:** Early versions of the model added components like Top-2 pitch quality, worst pitch OPS, and strikeout rate on top of W_OPS. All of these are wrong for the same reason: the sim selects pitches from a fixed probability distribution. You cannot instruct your pitcher to throw his fastball more. Because the usage percentages are fixed sim probabilities, W_OPS already captures every aspect of pitch quality — the best pitch is weighted by how often the sim calls it, the worst pitch is weighted by how often the sim calls it, and everything in between. Adding any pitch-specific component on top of W_OPS is double-counting. K/BB is the only genuinely independent signal because walks aren't captured in per-pitch OPS splits.

**No GB% for pitchers:** Ground ball rate is already embedded in W_OPS — a pitcher who generates grounders allows fewer hits and fewer home runs, which shows directly in his pitch OPS against. Adding GB% as a separate term double-counts it while penalizing strikeout pitchers who achieve the same W_OPS through a different approach. Both are valid pitcher archetypes; W_OPS correctly treats them equivalently at the same run prevention level.

---

## Roster Column Reference

The app expects the standard MLBC CSV export format. Key columns used:

**Hitters:** `Bat`, `Throw`, `Run`, `Arm`, `IF_Rng`, `OF_Rng`, `Fld`, `B_H`, `B_B2`, `B_B3`, `B_HR`, `B_BB`, `B_SO`, `vL_OBP`, `vL_SLG`, `vR_OBP`, `vR_SLG`, `Pull`, `B_GB`

**Pitchers:** `Throw`, `END`, `P_SO`, `P_BB`, `P1`–`P6`, `P1_Qual`–`P6_Qual`, `P1_vL_OBP`–`P6_vR_SLG`

**Both:** `Last`, `First`, `Age`, `Pos`, `Team`, `Level`

**Pro Years Report:** `Firstname` (or `First`), `Lastname` (or `Last`), `Pro Years`

---

## License

MIT License — see [LICENSE](LICENSE) for details. Free to use, modify, and share.

---

## Contributing

Pull requests welcome. Key areas for improvement:
- Catcher framing bonus (arm data exists but framing grade does not)
- Multi-position eligibility display
- Career trajectory tracking across seasons
- Draft board mode (A-ball only filter with pick slot context)
