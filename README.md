**Name(s)**: Michael Huang

---

## Introduction

### The dataset

This project uses the **2022 League of Legends competitive match dataset** from [Oracle's Elixir](https://oracleselixir.com/tools/downloads). In the raw file, every `gameid` has up to **12 rows**, one for each of the 10 players (a `top`, `jng`, `mid`, `bot`, and `sup` on each side) plus **2 team-summary rows**. Because my question is about a *team-level* strategic decision, I keep only the **team rows**, leaving the dataset with ~25000 rows for ~12500 games.

### My question

Early in a game, a team fights over neutral objectives, the **first Dragon** and the **Rift Herald**. Each objective gives different advantages (Dragon = long-term stacking buffs; Herald = a battering ram for early tower gold and map pressure). Players always debate how much getting the first Dragon actually matters. My central question is:

> **In professional games, is securing the first Dragon associated with winning the game?**

### Columns I use

After keeping team rows, the relevant columns are:

| Column | Description |
|---|---|
| `gameid` | Unique identifier for each game (2 team rows share one) |
| `league` | The professional league the game was played in (e.g. `LCK`, `LPL`) |
| `side` | `Blue` or `Red` side |
| `result` | **Response variable** — `1` if the team won the game, `0` if it lost |
| `firstdragon` | `1` if this team secured the game's first dragon, else `0` |
| `firstherald` | `1` if this team secured the game's first Rift Herald, else `0` |
| `firstblood` | `1` if this team drew first blood (first kill) |
| `firsttower` | `1` if this team destroyed the first tower |
| `firstbaron` | `1` if this team took the first Baron (a *late*-game objective) |
| `dragons`, `heralds` | Total dragons / heralds the team took all game |
| `golddiffat15` | This team's gold lead (or deficit) at 15 minutes |
| `gamelength` | Length of the game in seconds |
| `datacompleteness` | Whether Oracle's Elixir has `complete` or `partial` data for the game |

---

## Data Cleaning and Exploratory Data Analysis

**Cleaning steps and their impact.**

1. I filtered to `position == 'team'` to keep only the team rows. I kept a copy (`team_all`) that still contains missing values for the Assessment of Missingness step.
2. I selected relevant columns like the outcome (`result`), the early objectives (`firstdragon`, `firstherald`, `firstblood`, `firsttower`), context (`league`, `side`, `gamelength`, `golddiffat15`), and `datacompleteness`.
3. I dropped rows missing either `firstdragon` or `firstherald`, since they are only recorded for `complete` games. This reduced around 4000 rows from the dataset.
4. I collapsed 0/1 flags into a single categorical (`Both`, `First Dragon only`, `First Herald only`, `Neither`) that captures which early objective a team took first.

| league   | side   |   result |   firstblood |   firstdragon |   firstherald |   golddiffat15 |   gamelength |
|:---------|:-------|---------:|-------------:|--------------:|--------------:|---------------:|-------------:|
| LCKC     | Blue   |        0 |            1 |             0 |             1 |            107 |         1713 |
| LCKC     | Red    |        1 |            0 |             1 |             0 |           -107 |         1713 |
| LCKC     | Blue   |        0 |            0 |             0 |             1 |          -1763 |         2114 |
| LCKC     | Red    |        1 |            1 |             1 |             0 |           1763 |         2114 |
| LCKC     | Blue   |        1 |            0 |             1 |             0 |           1191 |         1972 |

<iframe src="assets/dist_gamelength.html" width="800" height="500" frameborder="0"></iframe>

Most pro games last **~25–35 minutes**, with a right skew toward longer games. This tells me the games are long enough that early objectives (dragon/herald in the first ~15 minutes) are only a small slice of the game, which can be used to examine if they influence the win rate.

<iframe src="assets/dist_golddiffat15.html" width="800" height="500" frameborder="0"></iframe>

`golddiffat15` is roughly symmetric. The spread (many games at 2,000+ gold) shows early leads are common and sizable to lead to winning games, motivating it as a feature for the prediction model.

<iframe src="assets/winrate_objective.html" width="800" height="500" frameborder="0"></iframe>

Teams that grab **both** early objectives win **~68%** of the time and teams that grab **neither** win only **~32%**, which means early tempo correlates with winning. The **First Dragon only (~49.5%)** and **First Herald only (~50.5%)** are near each other. That motivates controlling for one objective while testing the other.

<iframe src="assets/winrate_dragon_by_herald.html" width="800" height="500" frameborder="0"></iframe>

Within **each** herald group, getting the first dragon raises win rate by a similar amount (~+17 percentage points): from ~32% to ~50% without the herald, and from ~50% to ~68% with it. The dragon's effect looks **consistent regardless of the herald**, which is supporting evidence that overall the effect of herald can be controlled and the association between dragon and win can be tested in Step 4.

|                  |   No first herald |   Got first herald |
|:-----------------|------------------:|-------------------:|
| No first dragon  |              32.3 |               50.5 |
| Got first dragon |              49.5 |               67.8 |

Pivot table of the previous bar chart. Since the jump is nearly identical in both columns if a team gets the dragon, it shows that getting dragon is valuable and testable to see if it associates with winning.

---

## Assessment of Missingness

### MNAR reasoning

Several columns (`golddiffat15`, `firstherald`, `firstbaron`) are missing for the same games where `datacompleteness == 'partial'`. I do **not** believe `golddiffat15` is **MNAR**. A team's 15-minute gold lead isn't hidden because the lead was large or small, it is missing because the data provider did not collect a full in-game timeline for that match, which might be the case in smaller pro regions. Since the reason for missingness is an observable property of the game (`league`) rather than the gold value itself, this is better described as **MAR**. I also tested to show that it is not **MCAR**, because the missingness rate is very uneven across leagues (shown in the plot below).

**Additional data that would explain the missingness:** information about the data-collection pipeline. For example, which broadcasts/leagues had live API timeline access in 2022. With that, the missingness would be fully explained by observed columns, confirming MAR rather than MNAR.

<iframe src="assets/missingness_league.html" width="800" height="500" frameborder="0"></iframe>

The permutation test gives **p ≈ 0.00**, so I **reject** the null of no dependence: the missingness of `golddiffat15` **depends on `league`**. By observing the chart, among the 20 largest leagues, only **LPL** and **LDL** (the Chinese leagues, ~3,450 games combined) are missing `golddiffat15`, and they are missing it **100%** of the time, while every other major league has essentially none. I did a quick search and it shows that in 2022 those broadcasts did not share detailed timeline data. This is exactly MAR: missingness is explained by an observed column (`league`), not by the gold values themselves.

Repeating the same permutation test against `side` gives an observed TVD of **~0.00** and **p ≈ 1.00**, so I **fail to reject** the null: the missingness of `golddiffat15` does **not** depend on `side`. This makes sense because when a game's timeline is uncollected, *both* the Blue and Red team rows are missing, so the missingness is balanced across sides. This is my example of a column where the missingness does **not** depend.

---

## Hypothesis Testing

I test whether securing the **first dragon** is associated with winning, using a standard permutation test: under the null, the `firstdragon` label is meaningless, so shuffling it should not change the win-rate gap.

- **Null hypothesis ($H_0$):** A team's win rate is the **same** whether or not it secured the first dragon. Any observed gap is due to chance.
- **Alternative hypothesis ($H_1$):** Teams that secure the first dragon have a **different** win rate from teams that don't.
- **Test statistic:** the difference of mean in win rates, `mean(result | got dragon) − mean(result | no dragon)` (two-sided).
- **Method:** a **permutation test** — I randomly **shuffle the `firstdragon` labels** across all games, re-split into the two groups, and recompute the difference, repeating 10,000 times to build the null distribution.
- **Significance level:** $\alpha = 0.05$.

<iframe src="assets/step4_null.html" width="800" height="500" frameborder="0"></iframe>

**Conclusion.** The observed win-rate difference (**+0.155**) sits far outside the null distribution, giving **p ≈ 0.00 < 0.05**. I **reject the null hypothesis**: there is strong evidence that securing the first dragon is associated with a higher win rate.

---

## Framing a Prediction Problem

Predict whether a team will **win the game** (`result`) using only information available in the **early game**.

- **Type:** binary **classification** (win vs. loss).
- **Response variable:** `result`. It is the outcome the whole strategic question is about, and the classes are naturally balanced (~50/50), which keeps evaluation clean.
- **Features (known "at time of prediction"):** early-game columns only, `firstblood`, `firstdragon`, `firstherald`, `firsttower`, and `golddiffat15`. I exclude anything that happens after 15 minutes, such as `firstbaron` or full-game totals like `dragons` / `gamelength` because they would leak the outcome.
- **Evaluation metric:** **accuracy**, because the two classes are balanced so accuracy is not misleading here; I also check the confusion matrix.

---

## Baseline Model

**Baseline model.** A `DecisionTreeClassifier` wrapped in a single `sklearn` `Pipeline`, using two features:

- `firstdragon`: **nominal** binary 0/1 numeric, so no encoding.
- `golddiffat15`: **quantitative**; decision trees split on raw thresholds, so no scaling.

Using the shared 75/25 `train_test_split`, the unrestricted tree reaches **~90.0% training** but only **~66.7% test** accuracy. That large gap means the model is **overfitting**, which means it memorizes the training data rather than generalizing. So this is not yet a "good" model.

**Improvement plan (Step 7):** constrain the tree by tuning `max_depth` with `GridSearchCV` cross-validation, add the remaining early-game signals (`firstblood`, `firstherald`, `firsttower`), then encoding any nominal ones with `OneHotEncoder` inside a `Pipeline` and compare against a `RandomForestClassifier`.

---

## Final Model

**Newly engineered features.** Beyond the baseline, the final model engineers **two new features**, both inside a single `Pipeline`:

1. **`scaled_gold`:** `golddiffat15` passed through a `StandardScaler`. Putting gold on a standard scale keeps this important feature comparable to the binary objective flags and makes the model robust to its raw units.
2. **`n_early_objectives`:** a `FunctionTransformer` that sums the four early-objective flags into a 0–4 count. This is meaningful for the data-generating process: a team that secured *most* early objectives has snowballed the early game, and the single count gives the model a clean "overall early dominance" signal rather than four separate splits.

The four raw flags are passed through. All feature engineering and the model are packed in one `Pipeline` via a `ColumnTransformer`.

**Algorithm and selection:** I chose a `RandomForestClassifier` because the baseline's failure was *variance* (a single deep tree overfit), and a forest averages many trees to reduce it. **Before tuning**, I decided to search `max_depth`, `n_estimators`, and `min_samples_leaf`, using 5-fold `GridSearchCV`. The best hyperparameters were **`max_depth=5`, `n_estimators=100`, `min_samples_leaf=1`**, which is a shallow forest, confirming that capping depth was the key fix for overfit.

**Improvement over baseline** (same train/test split):

| Model | Test accuracy |
|---|---|
| Baseline (unrestricted tree, 2 features) | 0.667 |
| **Final (tuned random forest, engineered features)** | **0.741** |

The final model improves test accuracy by **~7 points** and, with capped depth, no longer overfits. Honestly, most of that gain comes from *regularizing* the model rather than the extra features — `golddiffat15` already carries most of the early-game signal, so the added flags and count contribute only a little on top.

<iframe src="assets/confusion_matrix.html" width="800" height="600" frameborder="0"></iframe>

---

## Fairness Analysis

Group X = short games (game length at or below the median, ~31 minutes) and Group Y = long games (above the median).

**Evaluation metric:** accuracy (the model's own metric).

- **Null hypothesis ($H_0$):** the model is fair and its accuracy is the **same** for short and long games; any difference is due to chance.
- **Alternative hypothesis ($H_1$):** the model's accuracy **differs** between short and long games.
- **Test statistic:** difference in accuracy (short − long).
- **Method:** permutation test shuffling the short/long labels, 10,000 reps, $\alpha = 0.05$.

<iframe src="assets/fairness_null.html" width="800" height="500" frameborder="0"></iframe>

**Conclusion.** Accuracy is **~86.0%** on short games but only **~62.4%** on long games, resulting in a gap of **+0.236** with **p ≈ 0.00**, so I **reject the null**: the model is **not** fair across game length. I expected this to happen because the model only sees early-game features, which say little about long, close games that swing on late-game decisions. It is typical that the longer a game goes, early advantages diminish and teams fight on more equal grounds for late-game objectives, depending more on players' personal performance. So the model is more accurate for short games since it is based only on early-game features.
