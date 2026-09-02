# Parental Advisory: Does It Help or Hurt?
### By Elii Lopez

A project for DSC 80 at UCSD analyzing explicit content and popularity in Spotify tracks.

## Introduction

In this project, I examined a dataset of 114,000 music tracks spanning 114 genres, collected from the Spotify Web API. Each row represents a single track and includes both audio characteristics computed algorithmically by Spotify (such as danceability, energy, and valence) and metadata like popularity and explicit content flags. This dataset is adapted from the Spotify Dataset 1921–2020 by Yamac Eren Ay, originally sourced via Kaggle. I additionally merged in `artists.csv`, which provides artist-level follower counts, joined on artist name after extracting each track's primary artist.

Within this dataset, I focus on five genres: hip-hop, pop, reggaeton, edm, and country. These genres were chosen because they are musically distinct from one another across multiple audio dimensions. They differ substantially in typical tempo, energy, acousticness, and production style (e.g. country's acoustic, guitar-driven sound vs. edm's synthetic, high-energy production), as well as in lyrical and cultural norms around explicit content. This distinctness matters because it helps avoid conflating genre-level differences with the effect I'm actually interested in. Reggaeton in particular is a genre I have a personal connection to, and one where explicit and mainstream content coexist closely.

Specifically, I investigate whether tracks flagged as explicit differ in popularity from non-explicit tracks within the same genre, controlling for genre so that any observed difference can't simply be explained by explicit content being concentrated in inherently more (or less) popular genres. This matters because it isn't obvious which way the effect should go: explicit tracks could underperform due to platform or radio restrictions, or outperform due to an authenticity/edginess appeal — and the answer likely differs from genre to genre.

The dataset, filtered to these five genres, contains 5,000 rows (4,977 after removing duplicate entries). The columns most relevant to this question are:
- `popularity` — Spotify's 0–100 popularity score for the track
- `explicit` — whether the track is flagged as containing explicit content
- `track_genre` — the genre label assigned to the track
- `followers` — the primary artist's follower count on Spotify, merged in from `artists.csv`

## Data Cleaning and Exploratory Data Analysis

**Data cleaning:** After filtering to my five genres, I dropped 23 true duplicate rows (same track_id tagged to the same genre twice). I kept tracks appearing under multiple genres, since Spotify's dataset intentionally tags some tracks to more than one genre, and each genre-context is analytically meaningful for my question. I parsed `release_date` (which came in several inconsistent formats) into a clean `release_year` column. I also discovered that ~35% of tracks had `popularity == 0`, which likely reflects insufficient streaming data rather than true unpopularity — I excluded these rows for analyses involving popularity, leaving 3,208 tracks.

Here is the head of my cleaned DataFrame:

| track_id | track_name | track_genre | explicit | popularity |
|---|---|---|---|---|
| 2wrJq5XKLnmhRXHIAf9xBa | 10,000 Hours (with Justin Bieber) | country | False | 78 |
| 6AHJTA1BN7ePfChCwqph3z | Country On | country | False | 0 |
| 5eUtyONoPyfZYGrFHmZzlc | Die A Happy Man | country | False | 1 |
| 1e3QZ42GsP8cTy5uQ0G7J3 | Something in the Orange | country | False | 3 |
| 5VnxOs7H73V3l6qPWvbHIM | Something in the Orange | country | False | 4 |


**Univariate analysis:**

<iframe
  src="assets/popularity_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Popularity is bimodal, with a large spike near 0 (likely tracks with insufficient streaming data) and a roughly normal distribution centered around 60–70 for the remaining tracks.

<iframe
  src="assets/energy_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Energy is left-skewed, peaking around 0.65–0.75, reflecting the fact that most of my chosen genres (edm, pop, reggaeton, hip-hop) lean energetic, with country pulling the distribution's tail toward lower energy.

**Bivariate analysis:**

<iframe
  src="assets/energy_valence_scatter.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

There's a weak positive relationship between energy and valence — higher energy tracks tend to sound slightly more positive — though genres overlap heavily rather than clustering distinctly, with country as an exception, skewing toward lower energy overall.

<iframe
  src="assets/popularity_by_genre_explicit.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This plot shows the core relationship I investigate throughout this project: the direction of the explicit/popularity relationship isn't consistent across genres. In pop, explicit tracks skew noticeably higher in popularity; in edm and hip-hop, explicit tracks trend slightly lower instead.

**Interesting aggregates:**

| track_genre | explicit | mean | median | count |
|---|---|---|---|---|
| country | False | 40.93 | 45.0 | 386 |
| country | True | 45.52 | 43.0 | 27 |
| edm | False | 55.24 | 63.0 | 563 |
| edm | True | 53.44 | 62.0 | 71 |
| hip-hop | False | 54.34 | 60.5 | 558 |
| hip-hop | True | 49.47 | 62.5 | 150 |
| pop | False | 57.60 | 67.0 | 756 |
| pop | True | 68.24 | 80.0 | 59 |
| reggaeton | False | 35.89 | 40.0 | 487 |
| reggaeton | True | 42.25 | 42.0 | 151 |

Explicit tracks show higher average popularity than non-explicit tracks in three of five genres (country, pop, reggaeton), while edm and hip-hop show the opposite pattern. This split direction is exactly why controlling for genre — rather than pooling across genres — was essential to this analysis.

## Assessment of Missingness

### MNAR Analysis

I considered whether any column in my dataset is likely NMAR (Not Missing At Random) — meaning missingness depends on the value itself, not on other observed columns.

`followers` (introduced after merging in `artists.csv`) is a plausible NMAR candidate. Missing values here likely stem from artist name mismatches during the join — differences in spelling, punctuation, or how featured/collaborating artists are credited between `tracks.artists` and `artists.name`. Since the true cause isn't fully captured by any column I observe, this leans toward NMAR, though it is also tied to observed variables like genre and explicit status, so a MAR argument is defensible as well. Obtaining an artist ID (rather than a name string) to join on instead would help resolve this ambiguity.

### Missingness Dependency

**Column 1: `tempo`** (880 missing out of 4,977 tracks)

- **Depends on `track_genre`** (observed TVD = 0.064, p = 0.003)
- **Does not depend on `popularity`** (observed difference in means = 1.61, p = 0.20)

This pattern is consistent with MAR: tempo is likely missing when Spotify's tempo-detection algorithm struggles with tracks lacking a clear, steady beat — a property correlated with genre, which I observe, rather than a fully hidden cause.

**Column 2: `followers`** (merged in from `artists.csv`, joined on artist name after deduplicating duplicate artist entries)

- **Depends on `track_genre`** (observed TVD = 0.273, p = 0.0)
- **Depends on `explicit`** (observed difference in proportions = 0.140, p = 0.001)
- **Does not depend on `popularity`** (observed difference in means = -6.66, p = 0.072)
- **Does not depend on `duration_ms`** (observed difference in means = 5693.8, p = 0.38)

Unlike `tempo`, `followers`' missingness is tied to genre and explicit status but not popularity or duration, suggesting the unmatched tracks cluster among specific genre/explicit-status combinations rather than being explained by how popular or long the track is. This is most plausibly a join artifact rather than a property of the tracks' audio or performance itself.

## Hypothesis Testing

**Null hypothesis (H0):** Within reggaeton, the mean popularity of explicit and non-explicit tracks is the same; any observed difference is due to random chance.

**Alternative hypothesis (H1):** Within reggaeton, the mean popularity of explicit and non-explicit tracks is different.

**Test statistic:** absolute difference in mean popularity between explicit and non-explicit tracks (two-sided, since I have no assumed direction).

**Significance level:** 0.05

**Result:** observed test statistic = 6.36, p-value = 0.027

**Conclusion:** Since p = 0.027 < 0.05, I reject the null hypothesis. This suggests that, within reggaeton specifically, explicit and non-explicit tracks do have different average popularity. This result does not prove causation or an absolute truth — it only indicates the observed difference is unlikely to have arisen by random chance under this permutation model.

## Framing a Prediction Problem

**Prediction problem:** Predict a track's `popularity` score (0–100) using its audio features, genre, explicit status, and artist follower count.

**Type:** Regression, since `popularity` is a continuous numeric score, not a category.

**Response variable:** `popularity`, chosen because it's the central variable of this entire project.

**Features used (all known at time of prediction):** `danceability`, `energy`, `valence`, `loudness`, `tempo`, `acousticness`, `speechiness`, `duration_ms`, `track_genre`, `explicit`, `followers` — all properties of the track and artist that exist before a track accumulates streams.

**Evaluation metric:** RMSE, chosen over MAE because it penalizes larger prediction errors more heavily. R² is reported alongside it for interpretability.

## Baseline Model

**Model:** Linear Regression, wrapped in an sklearn Pipeline.

**Features used:**
- `track_genre` — nominal categorical, one-hot encoded
- `danceability` — quantitative, used as-is

**Performance:**
- Train RMSE: 27.79
- Test RMSE: 28.43
- Test R²: 0.054

**Is this a good model?** Not yet — an R² of 0.054 means genre and danceability alone explain only about 5% of the variance in popularity. Train and test RMSE are nearly identical, showing the model isn't overfitting; it's simply underpowered given how few features it uses.

## Final Model

**New features engineered:**

- **`log_followers`** — a log-transform of artist follower count, since follower counts are heavily right-skewed and relative growth in audience matters more than absolute growth.
- **`explicit`** — a binary indicator tied directly to my research question from Steps 1–4.
- **`loudness` (scaled)** — standardized since loudness is on a very different numeric scale than my other features.

**Modeling algorithm:** Random Forest Regressor, chosen over Linear Regression because it can capture non-linear relationships and interactions between features.

**Hyperparameter tuning:** I tuned `max_depth` using GridSearchCV with 5-fold cross-validation, searching over [3, 5, 8, 10, 15, None]. I chose `max_depth` because it controls the bias-variance tradeoff for tree-based models. The best-performing value was `max_depth = 10`.

**Performance comparison:**

| Metric | Baseline | Final |
|---|---|---|
| Train RMSE | 27.79 | 15.88 |
| Test RMSE | 28.43 | 22.53 |
| Test R² | 0.054 | 0.405 |

The final model substantially improves over the baseline, with test R² increasing from 0.054 to 0.405. This suggests that artist reach (followers), explicit status, and loudness — combined with genre and danceability — meaningfully improve the model's ability to predict a track's popularity.

## Fairness Analysis

**Group X:** Explicit tracks (in the test set)
**Group Y:** Non-explicit tracks (in the test set)

**Evaluation metric:** RMSE, since the final model is a regression model.

**H0:** The model is fair — its RMSE for explicit and non-explicit tracks is roughly the same, and any observed difference is due to random chance.

**H1:** The model is unfair — its RMSE differs between explicit and non-explicit tracks.

**Test statistic:** absolute difference in RMSE between the two groups.

**Significance level:** 0.05

**Result:** observed RMSE difference = 3.88, p-value = 0.118

**Conclusion:** Since p = 0.118 > 0.05, I fail to reject the null hypothesis. There is no statistically significant evidence that the final model performs differently on explicit tracks compared to non-explicit tracks — a reassuring result given that explicit status was itself a central variable throughout this project.
