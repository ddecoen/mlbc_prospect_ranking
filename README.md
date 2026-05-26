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
| OBP Grade | 40% | `(OBP_vL × 0.25) + (OBP_vR × 0.75)` | On-base is the most predictive offensive rate stat |
| XBH Grade | 38% | `HR×4 + 3B×3 + 2B×2` | Extra base value mirrors the sim's own fantasy formula |
| K/BB Grade | 15% | `B_SO / B_BB` | Plate discipline — lower is better |
| GB% Grade | 4% | `B_GB` | Small power-style signal — low GB% = fly ball tendency |
| Pull Grade | 3% | `Pull%` | Small pull-power signal |

**Why GB% and Pull are small weights:** Early versions of the model weighted these at 12% and 8% respectively. This incorrectly penalized legitimate contact/gap hitters — a player who hits .845 OPS to all fields with a high GB% would score *lower* than a .803 OPS pull hitter. GB% and Pull are style signals, not outcome signals. OBP and XBH already capture the outcomes; GB% and Pull just add a small bonus for pull-power profiles without punishing other approaches.

**Hit Tool Floor — Contact Credibility:**

A poor hit tool creates two separate problems, both now modeled explicitly:

**1. Contact-credibility discount on OBP grade.** A player with B_H=155 cannot sustain a .390 OBP regardless of what his edit splits say — the splits assume a level of contact he can't produce. The OBP grade is discounted proportionally:

`OBP_G = OBP_G_raw × min(1.0, B_H / 170)`

So B_H=155 → OBP grade multiplied by 0.912 (9% discount). B_H=140 → 18% discount. B_H=170+ → no discount.

**2. Flat hit tool penalty** applied to the final overall score:

| B_H | Penalty | Notes |
|-----|---------|-------|
| ≥ 170 | 0 | No penalty — acceptable contact |
| ≥ 160 | −1 | Slightly below average |
| ≥ 150 | −3 | Genuine contact risk |
| ≥ 140 | −5 | Significant red flag |
| < 140 | −8 | Cannot profile as a hitter |

The old model used a single -1 penalty for B_H=155, which was far too lenient. A walk-heavy, no-contact player (the "Barry Bonds eye with no contact ability" profile) would score artificially high because the OBP grade trusted edit splits that the hit tool makes impossible to sustain in the sim.

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

Premium positions (CF, SS, C) have **athleticism-conditional bonuses** — the premium only applies if the player can actually man the position at an above-average level. A slow CF is really a corner outfielder; a poor-range SS is really a third baseman.

| Position | Condition | Bonus |
|----------|-----------|-------|
| C | Arm ≥ 6.0 | +3.0 |
| C | Arm ≥ 4.5 | +2.0 |
| C | Arm < 4.5 | +1.0 |
| SS | IF_Rng ≥ 5.0 | +2.0 |
| SS | IF_Rng ≥ 4.0 | +1.0 |
| SS | IF_Rng < 4.0 | +0.5 |
| CF | Run ≥ 5.5 | +1.5 |
| CF | Run ≥ 4.5 | +0.75 |
| CF | Run < 4.5 | 0 |
| 2B / 3B | — | +0.5 |
| RF / LF | — | 0 |
| 1B | — | −0.5 |

A slow CF with Run=3.95 gets the same 0 bonus as a left fielder — because that player is a left fielder who happens to be playing center. The premium is for the position played at a premium level, not just for the positional label in the roster file.

---

## Pitcher Model

### STUFF Score

The pitcher model is built on one core insight: **in a sim with fixed pitch probability distributions, W_OPS is a complete and sufficient quality metric.**

The sim engine selects pitches randomly according to each pitcher's usage percentages (`P1_Qual` through `P6_Qual`). You cannot instruct your pitcher to throw his fastball more often — the probabilities are fixed. This means:

- The **best pitch** is already weighted into W_OPS by how often the sim calls it
- The **worst pitch** is already weighted into W_OPS by how often the sim calls it
- **Pitch count** doesn't matter — a 3-pitch pitcher who dominates with all three is equal to a 5-pitch pitcher with the same W_OPS
- **Top-2 or bottom-1 metrics** double-count what W_OPS already captures

Walks are already captured in per-pitch OBP against, which flows directly into W_OPS. K/BB only appears as a **bonus for true outliers** — it does not penalize anyone.

| Component | Weight | Stat | Logic |
|-----------|--------|------|-------|
| W_OPS | 100% | Usage-weighted OPS against across all pitches | Complete pitch quality signal — captures best, worst, mix, and command simultaneously |

**Why K/BB is not in STUFF:** Walks are already reflected in per-pitch OBP against — a pitcher who walks batters has a worse W_OPS as a direct result. Penalizing K/BB on top of W_OPS double-counts the same walks and systematically underrates pitchers who get outs without strikeouts: groundball artists, soft-tossers, and deception-based arms. In a sim where a strikeout and a groundout are both just outs, K/BB is not an independent quality signal on top of W_OPS. A pitcher like Charlie Blanco — three pitches all under .720 OPS, 65% GB rate — should not be ranked at 555th because his K:BB ratio is unflattering. His pitch outcomes are what matter.

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
- END < 5.0 → Reliever penalty (−3 points)

The penalty was reduced from −5 to −3 because some high-quality relievers have genuine roster value as swingmen or high-leverage arms. A −5 gate was washing out pitchers who belong in the top 200 regardless of role.

**Elite Closer Bonus (+2.0):**

A reliever with true ace-level pitch quality AND elite command is a genuine asset regardless of endurance. If a pitcher meets all three conditions:
- END < 5.0 (reliever role)
- W_OPS < 0.660 (top-tier pitch quality — the best ~1% of the RP pool)
- K/BB > 3.0 (elite command)

…he receives a +2.0 bonus. This partially offsets the −3 reliever penalty, netting −1 overall — correctly placing an elite closer just slightly below an equivalent starter, rather than burying him. The W_OPS < 0.660 threshold was calibrated to the actual prospect pool; sub-.600 W_OPS relievers essentially don't exist because that level of stuff almost always comes with enough durability to start.

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

**GB% and Pull as small signals, not large weights:** GB% and Pull were originally weighted at 12% and 8% of HIT respectively. This created a systematic bias against contact/gap hitters — a player hitting .845 OPS to all fields would score lower than a .803 pull hitter because his high GB% and low Pull% dragged down his HIT score. GB% and Pull are *style* signals, not *outcome* signals. OBP and XBH already capture the outcomes. Reducing these to 4% and 3% means they add a small bonus for pull-power profiles without penalizing other legitimate hitting approaches.

**Conditional positional premiums:** The positional bonus should only apply when the player can actually play the position at a premium level. A CF with below-average speed (Run < 4.5) is really a corner outfielder playing center — giving him the same +1.5 bonus as a true center fielder with elite speed would systematically overrate him. The same logic applies to SS (gated on range) and C (gated on arm strength). Fixed bonuses based purely on the roster position label, regardless of whether the player has the tools to play it, produce rankings that no experienced scout would recognize.

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
