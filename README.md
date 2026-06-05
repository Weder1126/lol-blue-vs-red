# Blue Side, Red Side: Does Map Side Decide Pro League of Legends Games?

**By Weder Qin and Kevin Cui**

A project for DSC 80 at UCSD, using the 2022 [Oracle's Elixir](https://oracleselixir.com/tools/downloads) professional *League of Legends* match dataset.

---

## Introduction

Competitive *League of Legends* is one of the largest and longest-running e-sports in the world, with dozens of regional leagues feeding into an annual World Championship. Each professional match is played between two five-player teams, and one team is assigned the **Blue side** of the map while the other gets the **Red side**. Players, casters, and analysts have argued for years about whether the side you start on actually matters: the two sides have different objective spawns, a different drafting (pick/ban) order, and broadcasters routinely flash a "blue-side win rate" stat as if it were a law of nature. This project asks a simple question and tries to answer it defensibly:

> **In professional League of Legends matches from the 2022 season, is the win rate of teams on Blue side different from teams on Red side?**

Knowing the answer matters in the real world: leagues occasionally tweak drafting rules or map objectives between seasons to keep the game balanced, and anyone who models match outcomes has to decide whether "side" is a variable worth controlling for. After answering the descriptive question, we build a model that **predicts the winner of a game** from information known before it starts.

**The data.** The raw 2022 file has **150,348 rows and 165 columns**. Each *game* contributes 12 rows — five players per side plus one team-summary row per side — so for a question about team-level win rates we keep only the **team-summary rows** (`position == 'team'`). That leaves **25,058 team-game observations across 12,529 unique games**, perfectly balanced between Blue and Red by construction.

The columns most relevant to this project:

| Column | Meaning |
|---|---|
| `gameid` | Unique match identifier (one game = two team-rows) |
| `league` | Professional league (e.g. LCK, LPL, LEC, LCS) |
| `date` | Match start timestamp |
| `side` | `Blue` or `Red` — which side of the map this team played |
| `position` | Player role; we keep only `team`-aggregate rows |
| `teamname` | The organisation (used to build prior-form features) |
| `result` | Binary outcome: `1` if this team won, `0` if it lost |
| `gamelength` | Game duration in seconds |
| `patch` | Game-balance version the match was played on |
| `playoffs` | Whether the game was a playoff game |
| `golddiffat15` | Gold lead at 15 minutes (early-game proxy; studied for missingness) |

---

## Data Cleaning and Exploratory Data Analysis

We cleaned the data in a few deliberate, data-motivated steps:

1. **Kept only team-summary rows** (`position == 'team'`). The dataset stores both per-player and per-team rows; keeping all of them would count every game 12 times. This is the single most important step — it changes the unit of analysis from "a player in a game" to "a team in a game," which is what our win-rate question is about.
2. **Parsed `date` to a datetime.** This unlocks chronological ordering, which we later need to build *leakage-free* "prior form" features (a team's record can only use games that happened earlier).
3. **Dropped rows with a missing `side` or `result`.** These come from incomplete uploads on Oracle's Elixir; leaving them in would quietly bias the win-rate proportions.
4. **Engineered readable columns from existing ones:** `win` (an integer copy of `result`), `game_minutes` (`gamelength / 60`), and `kda` (`(kills + assists) / max(deaths, 1)`). These are *post-game* summaries used only for exploration — they are never fed to the predictive models, because knowing a game's kills would leak its outcome.

We intentionally did **not** impute missing values here: the missingness in columns like `golddiffat15` is itself informative and is studied in the next section.

The head of the cleaned, team-level DataFrame (a few relevant columns):

| gameid                | league   | date                | side   |   result |   game_minutes |      kda |
|:----------------------|:---------|:--------------------|:-------|---------:|---------------:|---------:|
| ESPORTSTMNT01_2690210 | LCKC     | 2022-01-10 07:44:08 | Blue   |        0 |        28.55   |  1.47368 |
| ESPORTSTMNT01_2690210 | LCKC     | 2022-01-10 07:44:08 | Red    |        1 |        28.55   |  9       |
| ESPORTSTMNT01_2690219 | LCKC     | 2022-01-10 08:38:24 | Blue   |        0 |        35.2333 |  0.625   |
| ESPORTSTMNT01_2690219 | LCKC     | 2022-01-10 08:38:24 | Red    |        1 |        35.2333 | 18.3333  |
| 8401-8401_game_1      | LPL      | 2022-01-10 09:24:26 | Blue   |        1 |        22.75   |  8       |

### Univariate analysis

Most professional games last roughly **25–35 minutes**, with a long right tail of marathon games that drag past 45 minutes. The distribution is unimodal and right-skewed — useful context for why `game_minutes` is a noisy, game-level quantity rather than something tied to one team.

<iframe src="assets/game_length_hist.html" width="820" height="440" frameborder="0"></iframe>

### Bivariate analysis

Our headline relationship is **win rate by side, split across the tier-1 leagues**. Blue side has the higher win rate in **9 of the 10** major leagues shown, ranging from a razor-thin edge in the LCK (50.7% vs 49.3%) to a striking 60.4% vs 39.6% gap in the PCS. The dashed line marks 50% — "no side advantage."

<iframe src="assets/winrate_by_league.html" width="900" height="500" frameborder="0"></iframe>

### Interesting aggregate

Pivoting win rate by `league` × `side` makes the Blue-minus-Red gap directly comparable across regions. Blue's advantage is present almost everywhere but varies enormously in size — tiny in the LCK, huge in the PCS, and actually slightly reversed in the VCS and LCS. This regional variation is exactly what motivates the formal hypothesis test below (does the *overall* gap survive once we account for chance?).

| league   |   Blue |   Red |   blue_minus_red |
|:---------|-------:|------:|-----------------:|
| PCS      |  0.604 | 0.396 |            0.209 |
| LPL      |  0.55  | 0.45  |            0.099 |
| CBLOL    |  0.539 | 0.461 |            0.078 |
| MSI      |  0.538 | 0.462 |            0.075 |
| LEC      |  0.535 | 0.465 |            0.07  |
| WLDs     |  0.523 | 0.477 |            0.045 |
| LJL      |  0.516 | 0.484 |            0.033 |
| LCK      |  0.507 | 0.493 |            0.015 |
| VCS      |  0.495 | 0.505 |           -0.009 |
| LCS      |  0.493 | 0.507 |           -0.013 |

---

## Assessment of Missingness

**NMAR analysis.** The most interesting missingness in the dataset is in `golddiffat15` (gold difference at 15 minutes), missing in about **15%** of team-rows. We do **not** believe this column is **NMAR** (Not Missing At Random). NMAR would require the chance of a value being missing to depend on the *unobserved value itself* — for example, gold leads being hidden specifically when they were unusually large. Reasoning about how the data is generated makes that implausible: Oracle's Elixir only records the 15-minute timeline for games whose detailed match history was scraped, and an entire game's timeline is either captured or not. In fact `golddiffat15` is missing in **exactly** the rows where `datacompleteness == 'partial'`, and whether a game is "partial" is decided by *which league / data source* produced it — not by the size of the gold lead. So the missingness is fully explained by other **observed** columns (`league`, `datacompleteness`), which makes it **MAR**, not NMAR. The extra piece of information that would "explain away" the missingness is simply the league's data-tracking tier, which we already observe. (If instead we could not see the league or completeness flag, and the splits stopped being reported whenever one team fell catastrophically behind, *that* would be a genuinely NMAR mechanism — but the data does not support it.)

**Missingness-dependency permutation tests.** We tested whether the missingness of `golddiffat15` depends on two other columns, using the **Total Variation Distance (TVD)** between the column's distribution among missing vs. non-missing rows as the test statistic, with a permutation test that shuffles the missingness indicator (since both candidate columns are categorical, TVD is the natural statistic).

- **Depends on `league`** — observed **TVD ≈ 0.99**, p ≈ **0.0000**. The league makeup of the missing rows is almost completely different from that of the non-missing rows (e.g. the LPL never reports `golddiffat15`, while many leagues always do). We reject the null of independence: the missingness clearly depends on `league`, confirming the MAR-on-league story.

<iframe src="assets/missingness_null.html" width="820" height="440" frameborder="0"></iframe>

- **Does *not* depend on `side`** — observed **TVD = 0.0**, p = **1.0**. Because completeness is assigned at the game level and every game contributes exactly one Blue and one Red team-row, the missing and non-missing groups are each split 50/50 across side. We fail to reject independence. This is reassuring: our Blue-vs-Red comparison is **not** confounded by differential missingness — we are never systematically missing more Blue rows than Red ones.

---

## Hypothesis Testing

We tested whether map side is associated with winning.

- **Null hypothesis $H_0$:** In the population of 2022 professional games, a team's win probability is the same regardless of side; the observed Blue–Red gap is due to chance.
- **Alternative hypothesis $H_1$:** Blue-side and Red-side teams have different win probabilities (two-sided — we did not pre-commit to a direction).
- **Test statistic:** the absolute difference in win rates, $T = \lvert \hat{p}_{\text{Blue}} - \hat{p}_{\text{Red}} \rvert$. With a binary outcome and two groups this equals the Total Variation Distance between the side-conditional outcome distributions, and it is the most interpretable summary of a "side advantage" for a reader.
- **Significance level:** $\alpha = 0.05$.

We ran a **permutation test** that shuffles the win/loss outcomes across all 25,058 team-rows and recomputes $T$ each time (10,000 shuffles), simulating a world where side and result are unrelated. The observed statistic is $T_{\text{obs}} = \lvert 0.5248 - 0.4752 \rvert = \mathbf{0.0496}$, and **0 of the 10,000 permutations** produced a gap that large, giving a p-value of approximately **0 (p < 0.0001)**.

<iframe src="assets/hypothesis_null.html" width="820" height="440" frameborder="0"></iframe>

Because the p-value is far below 0.05, we **reject the null hypothesis**. The data are strongly inconsistent with "side doesn't matter": Blue-side teams in 2022 pro play won about 5 percentage points more often than Red-side teams, and a gap this large is extremely unlikely to be a fluke. We are careful not to overclaim — a permutation test cannot *prove* that Blue side causes wins, and it cannot rule out confounders such as draft order; it only tells us the association is very unlikely under pure chance.

---

## Framing a Prediction Problem

We frame a **binary classification** problem: **predict `result`** (`1` = this team won, `0` = it lost) at the team-game granularity. This is the single most natural target in competitive e-sports — every pre-game betting market and in-broadcast win-probability graphic is answering the same `P(win)` question — and it is recorded directly in the data with no joining needed. Because every game has exactly one winner and one loser, the classes are **perfectly balanced (50/50)**, so we use **accuracy** as the primary evaluation metric (it is not distorted by class imbalance here) and report **F1** as a secondary sanity check.

Crucially, we respect the **"time of prediction."** A real forecaster only knows things available *before kickoff*: the two teams, their league, which side they're on, the patch, whether it's a playoff game, and each team's track record going into the match. We therefore train **only** on pre-game features and deliberately exclude every post-game statistic (kills, gold, `golddiffat15`, objective timings, game length), since those would leak the very outcome we are trying to predict.

---

## Baseline Model

Our baseline predicts `result` from **two features**, both **nominal**: `side` (2 categories) and `league` (55 categories). There are **0 quantitative** and **0 ordinal** features. Both are one-hot encoded with `OneHotEncoder(handle_unknown='ignore')` — producing **57 encoded columns** — and fed to a `LogisticRegression`, with every step wrapped in a single scikit-learn `Pipeline`. We evaluate generalization on a **20% held-out test set** (created once and reused by the final model so the comparison is apples-to-apples).

| | Accuracy | F1 |
|---|---|---|
| **Baseline (side + league)** | **0.5104** | 0.5127 |

Is this model "good"? **Not really — and that's expected.** It barely clears the 0.50 base rate, because `side` only carries the ~2.5-point edge we found above and `league` says almost nothing about which *particular* team wins a given game. The baseline is a deliberately honest floor: it uses the obvious categorical context and nothing else, leaving plenty of room for the engineered features in the final model to show their value.

---

## Final Model

The baseline ignores the biggest real driver of who wins a pro game — **team strength** — so we engineered two new features that capture it, both computed strictly from information available *before* each game:

1. **`team_prior_winrate`** — the team's win rate over **all of its earlier games this season**, via `result.shift(1).expanding().mean()` after sorting chronologically by `(date, gameid)`. The `.shift(1)` is what keeps it honest: it excludes the current game's own outcome, so the feature only ever sees the past. From a data-generating standpoint this is the single most informative thing we can know going in — stronger organisations win more consistently, and a team's track record is a direct proxy for the latent skill that actually decides matches but that `side` and `league` cannot express.
2. **`team_games_played`** — how many prior games the team has in the data (`cumcount()`). This is a *confidence* signal: it lets the model discount a `team_prior_winrate` estimated from only a game or two (a team's first-ever game has no history at all, and a 1–0 record is far weaker evidence than 40–20), and it weakly tracks being an established, deeper roster.

We also added the nominal `patch` (the balance version, which shifts the meta) and the boolean `playoffs` flag, on top of the baseline's `side` and `league`. The two numeric features are imputed (a team's first game has no prior win rate; we fill the neutral value 0.5, "no information," rather than any data-derived mean that could leak) and standardized; all categoricals are one-hot encoded — again, all inside one `Pipeline`.

**Algorithm and hyperparameter search.** We kept `LogisticRegression` and tuned the **inverse-regularization strength `C`** — chosen *before* tuning because one-hot encoding `league` and `patch` creates a wide, sparse feature space, so the amount of L2 shrinkage is the dominant bias/variance knob. Using **5-fold `GridSearchCV`** over `C ∈ {0.01, 0.1, 1, 10, 100}` (run on the training set only), the best value was **`C = 0.01`** (strong regularization, which makes sense given all those rare league/patch dummies). We then refit the tuned model on the whole dataset for the fairness audit.

| | Accuracy | F1 |
|---|---|---|
| Baseline (side + league) | 0.5104 | 0.5127 |
| **Final (+ prior form, patch, playoffs)** | **0.5874** | **0.6000** |

The final model improves test accuracy by **+7.7 percentage points** (0.5104 → 0.5874). The improvement is driven overwhelmingly by `team_prior_winrate` — recent form is simply far more predictive of a single game than side or region. We also confirmed there is **no leakage**: the model's training accuracy (0.588) is essentially identical to its test accuracy (0.587); had we forgotten the `.shift(1)` and let the current result leak in, training accuracy would have spiked toward 1.0.

<iframe src="assets/confusion_matrix.html" width="560" height="480" frameborder="0"></iframe>

---

## Fairness Analysis

Finally, we audited whether the final model works **as well for the biggest regions as for everyone else**.

- **Group X — "major":** games in the four marquee leagues (`LCK`, `LPL`, `LEC`, `LCS`).
- **Group Y — "other":** games in every other league.
- **Evaluation metric:** **precision** (of the games the model calls a win, how many truly were), which is comparable across groups of very different sizes; we also report recall.
- **Test statistic:** the absolute difference in precision, $\lvert \text{precision}_{\text{major}} - \text{precision}_{\text{other}} \rvert$.
- **Null hypothesis $H_0$:** the model is fair — precision is the same for both groups, and any gap is chance.
- **Alternative hypothesis $H_1$:** the model's precision differs between the two groups.
- **Significance level:** $\alpha = 0.05$.

| Group | n | Precision | Recall |
|---|---|---|---|
| major (LCK/LPL/LEC/LCS) | 3,604 | 0.5991 | 0.6104 |
| other | 21,454 | 0.5797 | 0.6081 |

The observed precision gap is **0.0194**. A permutation test that shuffles the major/other label across all rows (5,000 iterations) yields a p-value of **0.1168**.

<iframe src="assets/fairness_null.html" width="820" height="440" frameborder="0"></iframe>

Since 0.1168 > 0.05, we **fail to reject the null hypothesis**: there is no statistically significant evidence that the model's precision differs between major and other leagues. The model appears to treat the two groups comparably — the small edge for major leagues is well within what we'd expect from chance alone. As always with a hypothesis test, this is an absence of detectable unfairness on this metric, not a proof of perfect fairness.
