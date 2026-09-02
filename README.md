# Parental Advisory: Does It Help or Hurt?
### By Elii Lopez

A project for DSC 80 at UCSD analyzing explicit content and popularity in Spotify tracks.

## Introduction

In this project, I looked at a dataset of 114,000 music tracks across 114 genres, collected from the Spotify Web API. Each row is a single track, and includes both audio features that Spotify calculates automatically (like danceability, energy, and valence) and metadata like popularity and whether the track is explicit. The dataset is adapted from the Spotify Dataset 1921 to 2020 by Yamac Eren Ay, originally from Kaggle. I also merged in `artists.csv`, which has artist follower counts, joining on artist name after pulling out each track's main artist.

I focused on five genres: hip-hop, pop, reggaeton, edm, and country. I picked these because they sound pretty different from each other. They vary a lot in tempo, energy, acousticness, and production style (country's acoustic guitar sound versus edm's synthetic, high-energy production, for example), and they also have different norms around explicit lyrics. Picking distinct genres matters because if I compared genres that sound similar, it would be harder to tell whether differences came from the explicit label or just from genre overlap. Reggaeton is also a genre I have a personal connection to, and one where explicit and mainstream music sit close together.

My question is whether tracks marked explicit differ in popularity from non-explicit tracks within the same genre. Controlling for genre matters here. Without it, any difference I found could just reflect that explicit music is concentrated in genres that happen to be more or less popular overall. This question is genuinely open-ended. Explicit tracks might do worse because of platform or radio restrictions, or they might do better because some listeners find them more authentic, and the answer probably isn't the same in every genre.

Filtered to these five genres, the dataset has 5,000 rows (4,977 after removing duplicates). The columns that matter most for this question are:
- `popularity` — Spotify's 0 to 100 popularity score for the track
- `explicit` — whether the track is marked as explicit
- `track_genre` — the genre label Spotify assigned
- `followers` — the main artist's Spotify follower count, merged in from `artists.csv`

## Data Cleaning and Exploratory Data Analysis

**Data cleaning:** After filtering to my five genres, I dropped 23 rows that were true duplicates (the same track_id listed twice under the same genre). I kept tracks that showed up under multiple genres, since Spotify tags some songs to more than one genre on purpose, and each genre context is meaningful for my question. I parsed `release_date`, which came in a few different formats, into a clean `release_year` column. I also found that about 35% of tracks had a popularity of exactly 0, which probably means Spotify didn't have enough streaming data rather than the track being truly unpopular. I dropped those rows for any analysis involving popularity, which left 3,208 tracks.

Here's the head of my cleaned DataFrame:

| track_id | track_name | track_genre | explicit | popularity |
|---|---|---|---|---|
| 2wrJq5XKLnmhRXHIAf9xBa | 10,000 Hours (with Justin Bieber) | country | False | 78 |
| 6AHJTA1BN7ePfChCwqph3z | Country On | country | False | 0 |
| 5eUtyONoPyfZYGrFHmZzlc | Die A Happy Man | country | False | 1 |
| 1e3QZ42GsP8cTy5uQ0G7J3 | Something in the Orange | country | False | 3 |
| 5VnxOs7H73V3l6qPWvbHIM | Something in the Orange | country | False | 4 |

You can see the zero-popularity issue right in these first few rows, which is what led me to filter them out.

**Univariate analysis:**

<iframe
  src="assets/popularity_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Popularity has two humps. There's a big spike near 0, which is probably tracks without enough streaming data, and then a roughly normal-looking distribution centered around 60 to 70 for everything else.

<iframe
  src="assets/energy_distribution.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

Energy is skewed toward the high end, peaking around 0.65 to 0.75. That makes sense given that most of my genres (edm, pop, reggaeton, hip-hop) are pretty energetic, with country pulling the low end of the distribution.

**Bivariate analysis:**

<iframe
  src="assets/energy_valence_scatter.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

There's a weak positive relationship between energy and valence, so higher energy tracks tend to sound a bit happier. The genres overlap a lot rather than forming clear clusters, though country stands out by sitting lower on energy.

<iframe
  src="assets/popularity_by_genre_explicit.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

This is the main plot for my question. The explicit-popularity relationship doesn't go the same direction in every genre. In pop, explicit tracks are noticeably more popular. In edm and hip-hop, they trend slightly lower instead.

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

Explicit tracks have higher average popularity in three of the five genres (country, pop, reggaeton), but the opposite is true in edm and hip-hop. The fact that it goes both ways is exactly why I controlled for genre instead of pooling everything together.

## Assessment of Missingness

### MNAR Analysis

I thought about whether any column here is likely NMAR, meaning the missingness depends on the missing value itself rather than on other columns I can see.

`followers` is the best NMAR candidate. Those missing values came from the join with `artists.csv`, and they probably happen when artist names don't match exactly between the two files, whether from spelling, punctuation, or how featured artists get credited. Since the real cause is name formatting, which I can't observe directly, this leans NMAR. That said, it's also tied to columns I can see like genre and explicit status, so a MAR argument works too. Joining on an artist ID instead of a name string would clear this up.

### Missingness Dependency

**Column 1: `tempo`** (880 missing out of 4,977 tracks)

- **Depends on `track_genre`** (TVD = 0.064, p = 0.003)
- **Does not depend on `popularity`** (difference in means = 1.61, p = 0.20)

This looks like MAR. Tempo is probably missing when Spotify's beat detection struggles with tracks that don't have a clear, steady rhythm, and that's tied to genre, which I can observe.

**Column 2: `followers`** (merged in from `artists.csv`, joined on artist name after dropping duplicate artist entries)

- **Depends on `track_genre`** (TVD = 0.273, p = 0.0)
- **Depends on `explicit`** (difference in proportions = 0.140, p = 0.001)
- **Does not depend on `popularity`** (difference in means = -6.66, p = 0.072)
- **Does not depend on `duration_ms`** (difference in means = 5693.8, p = 0.38)

Unlike tempo, followers missingness is tied to genre and explicit status but not to popularity or duration. So the unmatched tracks cluster in certain genre and explicit combinations rather than being explained by how popular or long a song is. That points to a join issue rather than something about the tracks themselves.

## Hypothesis Testing

**Null hypothesis:** Within reggaeton, explicit and non-explicit tracks have the same average popularity, and any difference I see is just chance.

**Alternative hypothesis:** Within reggaeton, explicit and non-explicit tracks have different average popularity.

**Test statistic:** absolute difference in mean popularity between explicit and non-explicit tracks. I used absolute value because I wasn't assuming which group would be higher, just that they might differ.

**Significance level:** 0.05

**Result:** observed statistic = 6.36, p-value = 0.027

**Conclusion:** Since 0.027 is below 0.05, I reject the null hypothesis. This suggests that within reggaeton, explicit and non-explicit tracks do differ in average popularity. This doesn't prove anything about causation. It just means the difference I saw is unlikely to have come from random chance alone.

## Framing a Prediction Problem

**Prediction problem:** Predict a track's popularity score (0 to 100) from its audio features, genre, explicit status, and artist follower count.

**Type:** Regression, since popularity is a continuous number, not a category.

**Response variable:** `popularity`. I picked it because it's the variable my whole project centers on.

**Features:** `danceability`, `energy`, `valence`, `loudness`, `tempo`, `acousticness`, `speechiness`, `duration_ms`, `track_genre`, `explicit`, and `followers`. All of these are things you'd know about a track and artist before the song racks up any streams, so there's no leakage from the outcome I'm predicting.

**Evaluation metric:** RMSE. I chose it over MAE because it punishes big misses more heavily, and being way off on a few tracks seems worse than being slightly off on a lot of them. I report R² alongside it since RMSE alone is hard to interpret without knowing the scale.

## Baseline Model

**Model:** Linear Regression in an sklearn Pipeline.

**Features:**
- `track_genre` — nominal categorical, one-hot encoded
- `danceability` — quantitative, used as-is

**Performance:**
- Train RMSE: 27.79
- Test RMSE: 28.43
- Test R²: 0.054

**Is this good?** Not really. An R² of 0.054 means genre and danceability together explain only about 5% of the variation in popularity. The train and test RMSE are almost identical, so it isn't overfitting, it just doesn't have enough to work with.

## Final Model

**New features I added:**

- **`log_followers`** — a log transform of artist follower count. Follower counts are extremely skewed, since a handful of artists have millions while most have very few. Taking the log means going from 10k to 20k followers counts similarly to going from 1M to 2M, which better matches how audience size actually works.
- **`explicit`** — a simple yes/no flag. This is the variable at the center of my question, so it made sense to give the model access to it.
- **`loudness` (scaled)** — standardized, since loudness is measured in decibels on a totally different scale (roughly -60 to 0) than my other features, which mostly sit between 0 and 1.

**Model:** Random Forest Regressor. I picked it over Linear Regression because it can pick up on non-linear patterns and interactions between features, like loudness mattering differently depending on genre.

**Hyperparameter tuning:** I tuned `max_depth` with GridSearchCV using 5-fold cross validation, trying [3, 5, 8, 10, 15, None]. I picked `max_depth` because it controls how complex each tree gets. Too shallow and the model misses real patterns, too deep and it starts memorizing noise, which is a real risk with only about 3,200 rows. The best value was 10.

**Comparison:**

| Metric | Baseline | Final |
|---|---|---|
| Train RMSE | 27.79 | 15.88 |
| Test RMSE | 28.43 | 22.53 |
| Test R² | 0.054 | 0.405 |

The final model is a lot better, with test R² going from 0.054 to 0.405. Artist follower count, explicit status, and loudness clearly add real information about how popular a track ends up being, well beyond what genre and danceability alone could tell us.

## Fairness Analysis

**Group X:** Explicit tracks in the test set
**Group Y:** Non-explicit tracks in the test set

**Metric:** RMSE, since this is a regression model.

**Null hypothesis:** The model is fair. Its RMSE is about the same for explicit and non-explicit tracks, and any gap is chance.

**Alternative hypothesis:** The model is unfair. Its RMSE differs between explicit and non-explicit tracks.

**Test statistic:** absolute difference in RMSE between the two groups.

**Significance level:** 0.05

**Result:** observed RMSE difference = 3.88, p-value = 0.118

**Conclusion:** Since 0.118 is above 0.05, I fail to reject the null hypothesis. There isn't statistically significant evidence that my model does worse on explicit tracks than non-explicit ones. That's a good sign, especially since explicit status was both a feature in the model and the main variable I was studying all along.

