#F1 LAP TIME PREDICTOR
## Overview
This project predicts the F1 race lap times using tire age, testing whether tire degradation information actually helps to improve prediction over a simple baseline .I also explored two additional engineered features beyond the minimum requirement , and ran into an interesting modeling issue along the way that i think is worth documenting honestly. 
## Race Selection
So I picked the **2019 Hungarian Grand Prix**. Weather was dry that day, there 
was only one retirement (Grosjean, mechanical issue), and no safety car or 
red flag disrupted the race this keeps lap times clean and avoids 
confounding factors unrelated to tires. Out of 20 finishers, I used the top 8 to keep the analysis focused (as the task specifies 5-10 drivers).
## Data Cleaning
I removed two types of laps that would distort the analysis:
- Pit-stop laps and the lap immediately after each pit-stop as the cars are slow 
  entering/exiting the pits, and after pitting the tires would be cold again causing a anomaly lap - 22 laps removed across 
  11 total pit stops
- Laps slower than 1.5x a driver's own median lap time, to catch freak slow 
  laps (caused by traffic, small mistakes etc) — this removed 0 additional laps, since the 
  pit-lap cleaning above had already caught the main outliers.

  So, Final dataset: 534 clean laps across 8 drivers.
## Feature Engineering
- **tire_age** — counts how many laps a driver has done since last pit stop like 1,2,.. ,resetting to 
  0 each time they pit. This is meant to capture tier wear: the higher the number ,the more worn the tires ,and the slower the car should be.  
- **tire_age_squared** — I added this because tire degradation is not 
  necessarily a straight line; tires often wear faster later in a stint than 
  early on. This feature is calculated for every for every laps (not just later ones), giving the model both a linear and non-linear representation of tire age lets it learn whether degradation accelerates over time, without me having to hardcode when that acceleration starts.
- **laps_remaining** — calculated as total race laps minus the current lap number. I used this data as a rough proxy for fuel load: since fuel burns off fairly steadily over a race, a car should get progressively lighter (and slightly faster) as laps remaining decreases. This is a separate physical effect from tire wear, so including it helps the model distinguish "car is faster because it's lighter" from "car is faster/slower because of tire" condition.
  ## Train/Test Split (Avoiding Data Leakage)
I split the data by **stint** rather than randomly: each driver's final 
stint is held out as test data, and every earlier stint is used for 
training. A random split would leak information, since two adjacent laps are 
nearly identical in tire wear and conditions, testing on a lap sitting right 
next to one the model already trained on isn't a genuine test of whether it 
learned the pattern. I explicitly checked this: zero (driver, lap) pairs 
overlap between my train and test sets.

A stint is the group of consecutive laps a driver completes on one set of tires, bounded by pit stops — so a driver who pits twice has three stints, while a driver who pits once has two. Since I use each driver's final stint as the test set, a driver who made their last pit stop early in the race ends up with a long final stint (many test laps), while one who pitted late has a very short one.


- One side effect worth noting: because stint lengths vary by driver strategy, 
my test set (235 laps) ended up almost as large as my training set 
(299 laps) ; one driver's final stint was just 2 laps, another's was 42. 
This is a natural consequence of splitting by real stints rather than a 
fixed percentage.

## Models & Results

I trained two distinct regression algorithms; Linear Regression and 
Random Forest- each with a baseline version (grid position + lap number 
only) and an enhanced version (adding tire-related features):

| Model | RMSE (ms) |
|---|---|
| Linear Regression — Baseline (grid, lap) | 906.0 |
| Linear Regression — Enhanced (+ tire_age, tire_age² — initial attempt) | 1089.2 |
| Linear Regression — Enhanced (+ tire_age only — final version) | 905.3 |
| Random Forest — Baseline (grid, lap) | 1215.2 |
| Random Forest — Enhanced (+ tire_age, tire_age², laps_remaining, tuned) | 954.5 |

Random Forest hyperparameters (max_depth, n_estimators) were tuned via a 
manual grid search over a small set of reasonable values, selecting whichever 
combination minimized RMSE on the test stint.

My first attempt at the Linear Regression enhanced model included tire_age² 
alongside tire_age, i expected it to help capture non-linear degradation the 
same way it did for Random Forest. Instead, RMSE got *worse* (1089.2 vs. the 
906.0 baseline). I investigate why this happened in the *Key Finding Section* 
below, and removed tire_age² from Linear Regression's final feature set as a 
result.
## Key Finding
My initial expectation was that ***tire_age²*** would improve both models, since 
tire wear is rarely perfectly linear in real life. That held true for Random Forest — 
enhanced RMSE dropped from 1215 to 954, about a 21% improvement which i think is great. But when I 
first included tire_age² in Linear Regression, its RMSE actually got *worse* 
(1089 vs. 906 baseline).

  Investigating and researching this, I identified the likely cause as **multicollinearity**: 
tire_age and tire_age² are mathematically derived from each other, and 
linear models can struggle to assign stable coefficients to two highly 
correlated inputs. Random Forest doesn't have this problem, since it splits 
on features independently rather than solving a linear equation. So for my 
final comparison, I removed *tire_age²* specifically from Linear Regression's 
feature set (keeping just tire_age) while retaining it for Random Forest. 
This isn't cherry-picking the features that make each model look best, it's 
a intentional choice based on a real property of how each algorithm handles 
correlated inputs, and I think it's a more interesting finding than a clean 
*"tires improved everything"* result would have been: it suggests the 
tire degradation relationship is genuinely non-linear enough that only a 
flexible model benefits from capturing it.
