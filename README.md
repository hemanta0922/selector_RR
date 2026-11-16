# Most Reliable Players Under Pressure: SupeRR Selector Season 2, Round 1

A step-by-step Jupyter notebook for undergraduates. It does two jobs:

* **Part A (Steps 1 to 36):** understand the six cricket tables and how they connect. This is your original notebook, unchanged.
* **Part B (Steps 37 to 52):** define **pressure**, measure batters and bowlers under pressure against their usual level, rank the two top-10 lists, and test how trustworthy the ranking is.

---

## 1. What the problem asks

Build a data-driven system that measures how batters and bowlers perform in high-pressure situations compared with their usual level, then submit:

1. **Two ranked top-10 lists** (batters, bowlers) chosen in the Unstop dropdowns.
2. **A methodology report** covering: pressure definition, performance measurement, fair comparisons (sample size, eligibility, context, IPL vs T20I), evidence for the top 10, and reliability and limitations.

You must define pressure yourself and show that it separates pressure from ordinary situations. The notebook gives you a complete, reproducible baseline for every one of these requirements.

## 2. Files

```
pressure_players/
├── cricket_multi_dataset_relationship_analysis_with_pressure.ipynb   <- the notebook (already executed, outputs visible)
├── README.md                                                          <- this file
└── outputs/                                                           <- Part B result tables (CSV)
```

Put the notebook in the same folder as the six data files (`ball_by_ball.csv` or `ball_by_ball(1).csv`, `matches.csv`, `competitions.csv`, `teams.csv`, `playing_eleven.csv`, `competition_team_player_mapping.csv`). Then:

```
pip install pandas numpy matplotlib jupyter
jupyter notebook
```
Open the notebook and choose **Kernel, Restart Kernel and Run All** (about 1 to 2 minutes). Nothing needs scikit-learn. The original CSV files are never modified; results go to `cricket_analysis_outputs/`.

## 3. How the six tables connect

```
competitions (comp_id)                 <- IPL / T20I and season
   ├── matches (match_id, comp_id)     <- winner, toss, rain-revised target
   │      ├── ball_by_ball (match_id)  <- one row per delivery: striker_id, bowler_id, runs, wicket ...
   │      └── playing_eleven (match_id, team_id, player_id)
   ├── teams (comp_id, team_id)
   └── competition_team_player_mapping (comp_id, team_id, player_id)   <- player names
```

| Table | Rows | What it gives this problem |
|---|---:|---|
| `ball_by_ball` | 200,999 | Every delivery: the match situation and each player's outcome |
| `matches` | 883 | Winner (to learn win probability), rain-affected matches |
| `competitions` | 141 | Separates IPL from T20I (7 are "T20": IPL 2022 to 2026 plus two emerging-team events) |
| `competition_team_player_mapping` | 6,413 | Canonical names and IDs for the final lists |
| `playing_eleven` | 28,381 | Optional context (batting order, impact player) |
| `teams` | 353 | Team names |

Checks found in Part A: every ball-by-ball match exists in `matches`; every striker and bowler maps to the player table; 16 matches have no ball data (all abandoned); `bowl_speed` has 38% zeros and impossible values (not used); a few player names map to more than one ID, so **use `player_id`** when matching names.

## 4. The pressure definition in one idea

> **Pressure is how much the next ball can swing the match.**

For every ball of a chase (second innings, target known) the notebook:

1. computes `runs_needed`, `balls_left`, `wickets_in_hand`, `required_rate`;
2. estimates the chasing team's **win probability** with a logistic regression (plain NumPy);
3. computes **Pressure Index = closeness × √lateness**, where `closeness = 1 - |2p - 1|` (1 when the match is a coin flip) and `lateness` is the share of the innings gone;
4. labels the top 25% of balls **High**, the bottom 50% **Normal**, the rest **Medium**.

**Evidence that it works (Steps 39 to 41):**

| Check | Result |
|---|---|
| Win-probability calibration | predicted and actual win rates agree within about 1 percentage point in every band |
| Swing in win probability per ball | High 0.031 vs Normal 0.006, i.e. **4.8 times larger** |
| Average chase-win probability | High 0.49 (open contest) vs Normal 0.47 (mostly decided early or late) |

## 5. How players are rated

1. **Ball value:** how much better than the league average for the **same league (IPL, T20I, other) and over**. A batter loses `WICKET_COST` (10 runs) when out; a bowler gains it for a credited wicket and is charged for wides and no-balls.
2. **Two records per player:** `P` = value per 100 balls on **High** pressure balls; `U` = value per 100 balls on all other balls (the player's *usual level*).
3. **Blend, don't trust small samples:** `R = U + w × (P − U)`, `w = n_pressure / (n_pressure + K)`, with `K = 300`. Step 44 shows blending predicts unseen matches better than raw pressure numbers.
4. **Consistency:** share of pressure matches (at least 6 pressure balls) where the player's value was zero or better, smoothed as `(hits + 2) / (matches + 4)`.
5. **Eligibility:** at least 100 pressure balls, 8 pressure matches and 150 usual balls (48 batters and 70 bowlers qualify).
6. **Final score** = `0.75 × z(R) + 0.25 × z(hit rate)`.

## 6. Results from this run

**Top 10 batters under pressure:** Suryakumar Yadav, Heinrich Klaasen, Jos Buttler, Abhishek Sharma, Tim David, Mohammad Rizwan, Phil Salt, Cameron Green, Shreyas Iyer, Tristan Stubbs.

**Top 10 bowlers under pressure:** Sufiyan Muqeem, Mujeeb Ur Rahman, Mohammad Nawaz, Krunal Pandya, Abrar Ahmed, Matheesha Pathirana, Josh Hazlewood, Sunil Narine, Rashid Khan, Lockie Ferguson.

Full tables with `player_id`, ball counts, `P`, `U`, `R`, hit rate and IPL share are in `outputs/B_top10_*.csv`. These lists are the **output of this baseline method**. If you change a setting, re-run and use the new tables.

**Reliability findings (Steps 44, 48, 49), which belong in your limitations section:**

* Pressure performance is **noisy**: split-half correlation of raw pressure value is -0.07 (batters) and 0.27 (bowlers), while the usual record is 0.41 (batters) and 0.13 (bowlers).
* Blending with the usual record predicted unseen matches better than raw pressure for both roles.
* Top 10 stability: changing `K`, the consistency weight or the wicket cost keeps 7 to 10 of 10 names for batters and 8 to 10 for bowlers. Stricter eligibility (150 pressure balls) changes bowlers most (6 of 10 remain), because some top bowlers have only 100 to 150 pressure balls. **Ranks 7 to 10 are the least certain.**

## 7. Settings you can change (Steps 42 and 45)

| Setting | Default | Meaning |
|---|---:|---|
| `WICKET_COST` | 10 | Runs a wicket is worth in ball value |
| `K` | 300 | Blending strength; larger = trust the usual record more |
| `W_CONSISTENCY` | 0.25 | Weight of the consistency component |
| `MIN_PRESSURE_BALLS` / `MIN_PRESSURE_MATCHES` / `MIN_USUAL_BALLS` | 100 / 8 / 150 | Eligibility |
| High / Normal cut-offs | top 25% / bottom 50% of Pressure Index | Step 40 |

## 8. Limitations

1. Pressure is rated only in **chases** (the target is known). First-innings balls help estimate the usual level but are not pressure-rated.
2. Small samples: most players have 100 to 400 high-pressure balls.
3. No opposition-strength adjustment (a weak T20I attack counts like a strong IPL one).
4. Wicket cost and weights are judgement calls (tested in Step 49, not proven).
5. Rain-affected, no-result and super-over matches are excluded from the win-probability model.
6. The model is observational: a high rating shows a strong record under pressure, not a proven cause.

## 9. Glossary for first-time readers

* **Win probability:** the model's estimate that the chasing team wins from this exact situation.
* **Calibration:** does "30%" really win about 30% of the time?
* **Swing:** how much the win probability changes after one ball.
* **Shrinkage / blending:** pulling a noisy estimate towards a more reliable one.
* **z-score:** a value expressed as "how many standard deviations above the group average".
* **Split-half:** compare a result in odd-numbered matches with even-numbered matches to see whether it repeats.
* **Spearman correlation:** correlation of ranks (1 = same ordering, 0 = unrelated).

## 10. Before you submit

* Copy the rank order from Step 46 into the Unstop dropdowns, matching by `player_id` if a name looks ambiguous.
* Make sure the report tables match the lists you submit.
* Write the explanation in your own words and be able to defend every design choice (the notebook is the evidence, not the report).
