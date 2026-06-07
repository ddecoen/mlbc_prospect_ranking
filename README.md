# MLBC Prospect Ranker

A browser-based prospect ranking engine for the **Minor League Baseball Club (MLBC)** simulation game. Upload your league export files and get instant, data-driven prospect rankings — no server, no Python, no installs required.

🔗 **Live app:** https://ddecoen.github.io/mlbc_prospect_ranking

---

## Visual Design

Each prospect row displays three stacked grade bars, styled to match the sim's own player card layout:

**Hitters**

| Bar | Color  | What it measures                                             |
|-----|--------|--------------------------------------------------------------|
| FLD | Purple | Fielding score — position-weighted range, arm, speed, hands  |
| HIT | Green  | Offensive score — OBP, XBH, K/BB, GB%, Pull                 |
| OVR | Gold   | Overall (90% HIT + 10% FLD + all bonuses)                   |

**Pitchers**

| Bar | Color | What it measures                                                  |
|-----|-------|-------------------------------------------------------------------|
| END | Red   | Raw endurance value (shown as the actual END number, not a grade) |
| STF | Blue  | Stuff score — pure W\_OPS on the 20-80 scale                     |
| OVR | Gold  | Overall (STUFF + END gate + all bonuses/penalties)                |

All bars fill proportionally on the **20–80 grade scale**. The **GRADES** column shows underlying component grades. The **BONUSES** column shows pills for all active adjustments.

---

## How to Use

1. Export two CSV files from your MLBC sim:
   - `League_Roster_XXXX.csv` — full league player roster with all edits/grades
   - `Pro_Years_Report_XXXX.csv` — pro years report (filters to prospects with 0 pro years)
2. Open the app and upload both files
3. Rankings generate instantly in your browser — filter, sort, and export to CSV

---

## Model Philosophy

The model is built around one core principle: **grades are grades**. A player's edits represent their true ability ceiling regardless of what level they play at. No level-based adjustments are applied to scores.

The model separates hitters and pitchers completely, scoring each against the full non-ML prospect pool.

**Prospect definition:** Pro Years = 0 AND Age ≤ 25. Level is irrelevant — a player who broke camp with the ML club but has never played a full ML season is still a prospect.

---

## Hitter Model

### HIT Score (90% of overall)

| Component  | Weight | Stat                                 |
|------------|--------|--------------------------------------|
| OBP Grade  | 40%    | `(OBP_vL × 0.25) + (OBP_vR × 0.75)` |
| XBH Grade  | 38%    | `HR×4 + 3B×3 + 2B×2`                |
| K/BB Grade | 15%    | `B_SO / B_BB`                        |
| GB% Grade  | 4%     | `B_GB`                               |
| Pull Grade | 3%     | `Pull%`                              |

**Contact-credibility discount on OBP:**
`OBP_G = OBP_G_raw × min(1.0, B_H / 170)`

A player with B_H=155 cannot sustain a high OBP regardless of what his edit splits say. B_H=170+ gets full credit.

**Hit tool floor penalty (flat, applied to final overall):**

| B\_H  | Penalty |
|-------|---------|
| ≥ 170 | 0       |
| ≥ 160 | −1      |
| ≥ 150 | −3      |
| ≥ 140 | −5      |
| < 140 | −8      |

### FLD Score (10% of overall)

Each position uses position-appropriate defensive weights with range z-scored within position group.

**Throw hand constraint:** Left-handed throwers cannot play meaningful infield positions other than 1B. LH throwers coded at 2B/3B/SS use OF range as their range component.

### Fielding Playability Rules

A glove below the playability floor is a real roster problem — a player with no defensive home has significantly reduced value even with a strong bat.

**Fielding % floor: .975**

| Fielding % | Result |
|------------|--------|
| ≥ .975 | Playable — normal FLD scoring |
| .970–.974 + OPS ≥ 1.000 | **Generational bat exception** — plays trained position normally, full positional premium, shows `GEN-BAT` pill |
| .970–.974 + OPS < 1.000 | Unplayable |
| < .970 | Unplayable — no exceptions |
| Catcher (any %) | Arm strength gates the premium, not fielding % |

**When unplayable:**
- FLD grade forced to 0
- Positional premium stripped entirely
- No defensive home penalty applied (see below)
- Details column shows `FLD%:X.XXX⚠` in red

### No Defensive Home Penalty

Applied to all unplayable-glove non-catchers. A player with no position has meaningfully reduced value — they can only contribute via DH, and teams cannot build around a one-dimensional bat.

| OPS / OBP condition | Penalty |
|---------------------|---------|
| OPS ≥ .950 | 0 — elite bat, can DH and contribute |
| OPS ≥ .850 | −1.5 |
| OPS ≥ .750 OR OBP ≥ .380 | −3.0 — some tools, no power profile |
| Below all thresholds | −5.0 — nothing to offer without a position |

Shows as `NO-POS−X.X` pill in the BONUSES column.

### Positional Premiums

Premium positions (CF, SS, C) have athleticism-conditional bonuses. C and SS bonuses are additionally scaled by positional training rating (F_C, F_SS on 0–1 scale).

| Position | Condition                  | Raw Bonus | Scaling                  |
|----------|----------------------------|-----------|--------------------------|
| C (RH)   | Arm ≥ 8.0                  | +3.0      | × F\_C                   |
| C (RH)   | Arm ≥ 6.5                  | +2.0      | × F\_C                   |
| C (RH)   | Arm ≥ 5.0                  | +1.0      | × F\_C                   |
| C (RH)   | Arm < 5.0                  | 0         | —                        |
| C (LH)   | Any arm tier               | 50% of RH | × F\_C × 0.5             |

**LH throwing catchers** receive half the arm bonus of an equivalent RH catcher. Left-handed throwers have a mechanical disadvantage throwing to bases from behind the plate — the same principle that prevents LH throwers from playing 2B/3B/SS. A LH catcher with Arm 8.0 and F\_C=1.0 earns +1.5 instead of +3.0.

**LH catcher FLD scoring:** Because a LH catcher may not be able to fully man the position, their FLD component is scored using their best eligible non-C position rather than catcher weights. Eligible positions for LH throwers are 1B, LF, and RF (not CF, 2B, 3B, or SS). The model checks F\_1B, F\_LF, and F\_RF familiarity ratings and uses the highest-trained position; if none are trained it defaults to RF. The positional premium in the bonus column still reflects the C listing (with the 50% LH penalty applied). A gold FLD@RF (or FLD@1B / FLD@LF) label appears in the details column when this override is active.
| SS       | IF\_Rng ≥ 5.0 | +2.0      | × F\_SS |
| SS       | IF\_Rng ≥ 4.0 | +1.0      | × F\_SS |
| SS       | IF\_Rng < 4.0 | +0.5      | × F\_SS |
| CF       | Run ≥ 5.5     | +1.5      | —       |
| CF       | Run ≥ 4.5     | +0.75     | —       |
| CF       | Run < 4.5     | 0         | —       |
| 2B / 3B  | —             | +0.5      | —       |
| RF / LF  | —             | 0         | —       |
| 1B       | —             | −0.5      | —       |

### Other Bonuses

| Bonus        | Trigger                          | Points |
|--------------|----------------------------------|--------|
| True Power   | OPS ≥ 1.050                      | +3.0   |
| True Power   | OPS ≥ 1.000                      | +1.5   |
| True Power   | OPS ≥ 0.950                      | +0.5   |
| True Leadoff | OBP ≥ 0.390 + Run ≥ 5.5          | +2.0   |
| True Leadoff | OBP ≥ 0.380 + Run ≥ 5.0          | +1.0   |
| Bat Hand     | Switch                           | +1.0   |
| Bat Hand     | Left                             | +0.5   |

---

## Pitcher Model

### STUFF Score

Pure W\_OPS — usage-weighted OPS against across all pitches. Complete pitch quality signal that already captures best pitch, worst pitch, mix, and command simultaneously.

**Batter handedness weights:**
- LHP faces 76.4% RHH → wR=0.764, wL=0.236
- RHP faces 64.7% RHH → wR=0.647, wL=0.353

**W\_OPS grade bins:**

| Grade | W\_OPS threshold |
|-------|-----------------|
| 80    | ≤ 0.575         |
| 75    | ≤ 0.595         |
| 70    | ≤ 0.615         |
| 65    | ≤ 0.640         |
| 62    | ≤ 0.660         |
| 58    | ≤ 0.680         |
| 54    | ≤ 0.710         |
| 50    | ≤ 0.745         |
| 45    | ≤ 0.790         |
| 40    | ≤ 0.850         |
| 35    | > 0.850         |

Values ≤ 0.575 get flat grade 80 (no smoothing — Koufax tier).

### END Gate

- END ≥ 5.0 → SP, no adjustment
- END < 5.0 → −3 (reliever penalty)

### Walk Penalty (context-aware)

| P\_BB | Base Penalty |
|-------|-------------|
| ≥ 55  | −1.0        |
| ≥ 60  | −1.5        |
| ≥ 65  | −1.75       |
| ≥ 70  | −2.0        |
| > 75  | −3.0        |

Multiplied by context factor: H+BB < 170 → ×0.25 / H+BB 170–200 → ×0.50 / H+BB > 200 → ×1.0

### Pitcher Bonuses (no bonuses when STUFF = 80)

| Bonus        | Trigger                                        | Points |
|--------------|------------------------------------------------|--------|
| SP quality   | STUFF ≥ 75, SP                                 | +3.0   |
| SP quality   | STUFF ≥ 70, SP                                 | +2.0   |
| SP quality   | STUFF ≥ 65, SP                                 | +1.5   |
| SP quality   | STUFF ≥ 60, SP                                 | +1.0   |
| Elite closer | RP + W\_OPS < 0.660 + K/BB > 3.0              | +2.0   |
| Elite BP arm | END 4.0–4.9 + W\_OPS ≤ 0.680 + K/BB > 2.5    | +2.0   |
| Elite K/BB   | K/BB > 4.0 (any role)                          | +2.0   |

---

## Roster Column Reference

**Hitters:** `Bat`, `Throw`, `Run`, `Arm`, `IF_Rng`, `OF_Rng`, `Fld`, `B_H`, `B_B2`, `B_B3`, `B_HR`, `B_BB`, `B_SO`, `vL_OBP`, `vL_SLG`, `vR_OBP`, `vR_SLG`, `Pull`, `B_GB`

**Pitchers:** `Throw`, `END`, `P_SO`, `P_BB`, `P1`–`P6`, `P1_Qual`–`P6_Qual`, `P1_vL_OBP`–`P6_vR_SLG`

**Both:** `Last`, `First`, `Age`, `Pos`, `Team`, `Level`

**Pro Years Report:** `Firstname` (or `First`), `Lastname` (or `Last`), `Pro Years`

---

## License

MIT License — see [LICENSE](LICENSE) for details.
