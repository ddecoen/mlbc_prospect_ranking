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

Built entirely from **per-pitch weighted OPS against**, using actual batter handedness distribution from the league:

- **LHP** faces 76.4% right-handed batters (R + switch batting right vs LHP)
- **RHP** faces 64.7% right-handed batters

Each pitch is weighted by its usage percentage (the `P1_Qual` through `P6_Qual` columns represent the fraction of pitches thrown, e.g. 0.23 = 23%). The model rewards pitchers who **throw their best pitches most often**.

| Component | Weight | Stat | Logic |
|-----------|--------|------|-------|
| W_OPS | 40% | Usage-weighted OPS against all pitches | Overall quality signal — what batters actually do |
| Top2_W | 30% | Weighted avg OPS of the two most-thrown pitches | Rewards leaning on best stuff; fixes multi-pitch count bias |
| Best_OPS | 15% | OPS against the single best pitch | Ace weapon ceiling — the pitch that gets big outs |
| K/BB | 10% | `P_SO / P_BB` | Command and control — pitcher-controlled outcomes |
| K Rate | 5% | `P_SO` | Strikeouts are the only truly pitcher-controlled out |

**Why strikeouts over ground balls:** A strikeout cannot become a hit, error, or double play ball gone wrong. Ground ball outs depend on defense, park, and luck. GB% is already implicitly captured in W_OPS (a ground ball specialist allows fewer hits, which shows in P_H and P_HR). Rewarding GB% separately double-counts it while disadvantaging legitimate strikeout pitchers.

**Endurance (END) as a gate only:**
- END ≥ 5.0 → Starter-eligible, no adjustment
- END < 5.0 → Reliever penalty (−5 points)

Endurance affects *volume* of contribution, not *quality*. A pitcher with END=5.5 and elite stuff is more valuable than one with END=8.0 and mediocre stuff. The gate prevents relievers from being overrated relative to starters but does not otherwise favor workhorse arms over quality arms.

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

**Per-pitch OPS with usage weighting:** The sim's pitch quality is reflected in per-pitch OPS against (each pitch has vL and vR OPS splits). A pitcher who throws his best pitch 50% of the time is more effective than one who buries it at 15% usage. The Top2_W component specifically rewards this — it finds the two most-thrown pitches and scores their quality, rewarding pitchers who lean on their best weapons.

---

## Roster Column Reference

The app expects the standard MLBC CSV export format. Key columns used:

**Hitters:** `Bat`, `Throw`, `Run`, `Arm`, `IF_Rng`, `OF_Rng`, `Fld`, `B_H`, `B_B2`, `B_B3`, `B_HR`, `B_BB`, `B_SO`, `vL_OBP`, `vL_SLG`, `vR_OBP`, `vR_SLG`, `Pull`, `B_GB`

**Pitchers:** `Throw`, `END`, `P_SO`, `P_BB`, `P_GB`, `P1`–`P6`, `P1_Qual`–`P6_Qual`, `P1_vL_OBP`–`P6_vR_SLG`

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
