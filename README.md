# NBA Tanking Watch Index

Justine Murdock, Owen Bao, Justin Che

MIT 15.285 Sports Strategy & Analytics, Spring 2026.
 
As a statistical detection of intentional tanking in the NBA, the project asks a question relevant to the league office: **can on-court and roster-deployment data distinguish a team that is *trying to lose* from a team that is simply bad?**

Rather than relying on win totals alone (which can't separate intent from incompetence), the project builds team-season features around the **All-Star break** (the point in the season when teams with no playoff hopes typically begin resting/shutting down veterans and playing younger players), and tests whether the *change* in effort, performance, and roster usage after the break predicts tanking better than record alone.

## Key question

> Can we statistically distinguish intentional tanking from legitimate losing using publicly available data?

## Approach

1. **Data collection** (`nba_api`): team game logs, hustle stats (deflections, contested shots, loose balls, charges drawn), standings, roster age, and individual player availability, for every team across **10 seasons (2016-17 through 2025-26)**, split into pre-/post-All-Star-break windows.
2. **Ground truth labels**: a hand-coded set of known tanking team-seasons (e.g., Philadelphia's "Process," post-trade rebuild years in Houston/OKC/Detroit/San Antonio/Utah/Washington) based on public front-office statements, documented veteran teardowns, media consensus, and historically extreme losing.
3. **Vegas preseason win totals** (sourced from Covers.com's NBA odds history): used to compute **underperformance** = actual wins minus Vegas-implied wins, a market-based expectation baseline independent of the model's own features.
4. **Feature engineering**: pre/post All-Star deltas in:
   - Point differential and win rate
   - Hustle stats (deflections, contested shots, loose balls recovered, charges drawn)
   - Minutes concentration among top players (`DELTA_MIN_CONC`)
   - Rookie/young-player minutes share (`DELTA_ROOKIE_MIN_SHARE`)
   - Star player inactive ("rest") rate, isolated from "Injury/Illness" absences (`DELTA_INACTIVE_RATE`)
5. **Logistic regression as a signal-validation step**: fit on the full 300 team-season sample to test which of the candidate features actually moved the needle on tanking probability, rather than assuming the four headline signals mattered just because they made narrative sense. This step both confirmed suspicions and corrected them: it surfaced `DELTA_ROOKIE_MIN_SHARE` (1.28x odds ratio) and `DELTA_INACTIVE_RATE` (1.20x) as predictors in the expected direction, but also flagged `DELTA_LOOSE_BALLS_RECOVERED` as the single largest coefficient (1.66x) for a confounded reason (rookies hustle for loose balls more than veterans), so it was deliberately left out of the index rather than taken at face value. Two signals that weren't significant by odds ratio, `DELTA_PM` and `DELTA_MIN_CONC`, showed large raw gaps between tanking and non-tanking teams and were kept in the index on that basis.
6. **Tanking Watch Index (TWI) construction**: a composite 0-100 score (`RobustScaler`-normalized, outlier-clipped) built from the four signals the logistic regression step validated, point-differential drop, minutes-concentration drop, rookie-minutes increase, and inactive-rate increase, weighted 40% minutes concentration / 20% each for the rest, with a sensitivity analysis confirming the ranking is stable across alternative weighting schemes.
7. **K-means clustering as an independent check**: clustering the same four TWI components with no ground-truth labels involved tests whether those signals reflect real structure in the data, rather than artifacts of how the labels were hand-coded. It recovers a high-risk cluster (37% ground-truth tank rate vs. ~3% elsewhere) that captures 72% of known tankers without ever being told which teams were tanking, which is the validation result the index leans on most.
8. **Visualization & case studies** for known tanking franchises (PHI, OKC, UTA, DET, NOP, WAS) plus league-wide trend charts (e.g., blowout rate by season, which hit a record 34% in April 2026).

## Sample results

The clustering step in point 7 above is the project's strongest piece of evidence: it independently regroups team-seasons using only the four TWI signals, with no ground-truth labels involved, and still finds a cluster where tanking is roughly 10x more common than in the rest of the league:

![Cluster scatter](outputs/fig_cluster_scatter.png)

## Repository structure

```
.
├── NBA_tanking_metric.ipynb        # Full pipeline: data collection, ground truth +
│                                    # Vegas labels, features, logistic regression,
│                                    # TWI index, clustering, charts
├── NBA Tanking Memo + Technical Appendix.pdf            # Write-up for a league-office
│                                                          # audience, summarizing findings
├── NBA Tanking Memo + Technical Appendix (with code).pdf # Same memo with the supporting
│                                                          # code included
├── requirements.txt
├── hustle_data/                    # Cached pre/post All-Star hustle-stat features
├── player_availability/            # Cached pre/post All-Star player availability features
│                                    # by season (2016-17 through 2025-26)
├── outputs/                        # All generated figures, scores, and intermediate/
│                                    # final CSVs (game logs, standings, model results,
│                                    # TWI scores, cluster assignments)
└── NBA_tanking_hot_take_video.mp4   # Short video summary (not tracked in git, see below)
```

## Getting started

```bash
pip install -r requirements.txt
```

Open `NBA_tanking_metric.ipynb` in Jupyter and run top to bottom. The first run will hit `nba_api` for all 10 seasons (rate-limited, so expect it to take a while); cached intermediate features are written to `outputs/`, `hustle_data/`, and `player_availability/` so subsequent analysis cells don't require re-fetching.

All figures and result tables are written to `outputs/`.

## Data sources

- **[nba_api](https://github.com/swar/nba_api)**: official NBA Stats endpoints (game logs, hustle stats, standings, roster/team stats, player availability)
- **Covers.com Sports Odds History**: Vegas preseason win totals, hand-entered into the notebook for the underperformance feature

## Note on the video

`NBA_tanking_hot_take_video.mp4` (~227MB) is excluded from version control via `.gitignore` because it exceeds GitHub's 100MB per-file limit. It's available locally in this directory.
