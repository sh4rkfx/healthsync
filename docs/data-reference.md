# Data reference

All files live in `data/` (or `HEALTHSYNC_DATA_DIR`). Locally that is the clone of the private
data repository; in the data repository itself they sit at the root. CSV files are UTF-8 with a
header row; empty fields mean the value was not available.

## garmin_daily.csv

One row per calendar day. Overnight values belong to the night that ends on this date.

| Column | Unit | Notes |
|---|---|---|
| `date` | YYYY-MM-DD | Local calendar day |
| `weight_kg`, `fat_mass_kg`, `muscle_mass_kg`, `bone_mass_kg` | kg | First weigh-in of the day. `fat_mass_kg` is derived from weight and body fat % |
| `body_fat_pct`, `body_water_pct` | % | Bioimpedance estimates, use as weekly trends only |
| `bmi` | kg/m² | |
| `sleep_score` | 0–100 | |
| `sleep_hours` | h | Total sleep time |
| `sleep_start_local`, `sleep_end_local` | YYYY-MM-DD HH:MM | Falling asleep and waking up in `HEALTHSYNC_TZ`. On travel days this is home time |
| `deep_sleep_min`, `light_sleep_min`, `rem_sleep_min`, `awake_min` | min | Sleep stages |
| `hrv_last_night_ms`, `hrv_weekly_avg_ms` | ms | Overnight HRV average and Garmin's 7-day average |
| `hrv_status` | text | e.g. `BALANCED`, `UNBALANCED`, `LOW` |
| `hrv_baseline_low_ms`, `hrv_baseline_high_ms` | ms | Garmin's personal balanced range |
| `resting_hr` | bpm | |
| `body_battery_wake` | 0–100 | At wake-up, if Garmin provides it |
| `body_battery_high`, `body_battery_low` | 0–100 | Daily maximum and minimum |
| `body_battery_charged`, `body_battery_drained` | points | Gained and used during the day |
| `stress_avg` | 0–100 | Empty if Garmin had too little data |
| `steps` | count | |
| `calories_total`, `calories_active` | kcal | Garmin's estimate of energy burned: total (resting + active) and active only |
| `readiness_score` | 0–100 | Morning value; Garmin recalculates readiness after activities |
| `readiness_level` | text | e.g. `HIGH`, `MODERATE`, `LOW` |

## garmin_activities.csv

One row per activity recorded on the watch.

| Column | Unit | Notes |
|---|---|---|
| `date`, `start_time_local` | | Local date and time where the activity took place |
| `activity_id` | | Garmin Connect ID |
| `type` | text | Garmin type key, e.g. `bouldering`, `hiking`, `gravel_cycling`, `running`, `strength_training` |
| `name` | text | Activity name as shown in Garmin Connect |
| `duration_min`, `moving_min` | min | Total and moving time |
| `distance_km` | km | Empty for activities without distance |
| `elevation_gain_m` | m | |
| `avg_hr`, `max_hr` | bpm | |
| `calories` | kcal | |
| `training_load` | points | Garmin's EPOC-based load, comparable across sports |
| `aerobic_te`, `anaerobic_te` | 0–5 | Training Effect |
| `location` | text | If Garmin provides it |

Sport-specific details such as climbing routes are not part of the activity list. `dump-raw`
saves the full details of a day's activities if you want to add such fields.

## hevy_sets.csv

One row per set.

| Column | Notes |
|---|---|
| `date` | Local calendar day of the workout start |
| `start_time` | Workout start, UTC, ISO 8601 |
| `workout_id`, `workout_title` | Title as set in Hevy, used by the dashboard to compare workouts |
| `exercise_index`, `exercise` | Position and name within the workout |
| `exercise_template_id` | Stable exercise ID; unlike the name it survives renames |
| `set_index`, `set_type` | `set_type` is `normal`, `warmup`, `dropset` or `failure`. Everything except `warmup` counts as a hard set |
| `weight_kg`, `reps`, `rpe` | |
| `duration_s`, `distance_m` | For timed or cardio sets |

## nutrition_daily.csv

Optional, from YAZIO (unofficial app API). One row per day with at least one food or water entry;
days without entries have no row. Calories, protein, carbohydrates and fat are the sums over all
meals of the day.

| Column | Notes |
|---|---|
| `date` | Local calendar day |
| `calories_kcal` | Energy intake. Empty on days with water entries but no food |
| `protein_g`, `carbs_g`, `fat_g` | Grams per day, rounded to 0.1 |
| `water_ml` | Logged water intake |

A partly logged day is indistinguishable from a day of low intake. Check the mapping against your
data with `python -m healthsync dump-yazio --date <day>` after the first login.

## insights.json

Written by `python -m healthsync analyze`. Method: [analysis.md](analysis.md).

| Field | Content |
|---|---|
| `generated_at` | UTC timestamp of the run |
| `period` | `start`, `end` and `days` of the analysis window |
| `method` | Short description of correlation, effect size and correction |
| `coverage` | Days with a value per variable inside the window |
| `tested`, `skipped_too_few_days`, `robust` | Counts of pairs |
| `findings` | One entry per tested pair, sorted by status, q and strength |

Each finding:

| Field | Example | Content |
|---|---|---|
| `driver`, `driver_label` | `bedtime_min`, "Bedtime" | What varies first |
| `outcome`, `outcome_label` | `hrv`, "Overnight HRV" | What is affected |
| `lag_days`, `timing` | `0`, `same_night` | `same_night`, `next_night`, `second_night` or `same_day` |
| `n` | `175` | Days used |
| `rho` | `-0.32` | Spearman correlation of the detrended values, -1 to 1 |
| `effect`, `effect_unit`, `effect_per` | `-2.0`, `ms`, "1 hour later" | Change of the outcome per step of the driver |
| `p`, `q` | `0.0001`, `0.001` | Uncorrected and Benjamini-Hochberg corrected p-value |
| `status` | `robust` | `robust`, `tendency` or `none` |

Variable keys: `training_load`, `active_min`, `steps`, `hard_sets`, `tonnage_t`, `stress_avg`,
`bedtime_min` (minutes relative to midnight, negative before midnight), `sleep_hours`,
`sleep_score`, `hrv`, `resting_hr`, `readiness`, `body_battery`, `strength_form_pct`.
