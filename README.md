# Temporal Deviation Modeling for Early Identification of Depressive Symptom Worsening: A Cross-Dataset Analysis of Smartphone and Wearable Signals

---

## StudentLife ETL & Feature Extraction Pipeline

This document explains what each script in this repository does, the methodology for deriving sensing-based attributes, and how to run each component of the ETL pipeline.

The goal of this pipeline is to transform raw StudentLife sensing data into structured **per-day behavioural features** for the research project.  
In addition to the original ETL and per-day feature extraction, we have now **extended the pipeline** with:

- Per-day **audio** and **WiFi** context features (`audio_per.py`, `wifi_per.py`)
- A full set of **change-point detection scripts** for each sensing domain (sleep, mobility, activity, social, audio, WiFi)
- Helper scripts to **inspect** and **summarise** change points stored in MongoDB

---

## 1. `etl.py` — Universal StudentLife Importer

**Source:** `etl.py`

### What it does

- Reads *all* StudentLife sensing directories and CSV files.
- Resolves formatting inconsistencies (missing columns, corrupted rows).
- Detects and injects `participant_id` from filename.
- Normalizes types (timestamps, floats, NaN handling).
- Creates one MongoDB collection per sensing directory.
- Ensures every raw row is imported safely.

### How to run

```sh
python etl.py ./path/to/sensing/
```

**Example**

```sh
python etl.py ./dataset/sensing/
```

---

## 2. `activity_per.py` — Daily Physical Activity Features

**Source:** `activity_per.py`

### What it does

Aggregates StudentLife’s activity inference sensor into daily summaries stored in `per_activity`.

### Methodology

StudentLife provides minute-level inference values:

- `0` = stationary  
- `1` = walking  
- `2` = running  
- `3` = unknown / noise

### Extracted attributes

| Attribute           | Description                         |
|---------------------|-------------------------------------|
| walking minutes     | Count of samples with inference = 1 |
| running minutes     | Count of samples with inference = 2 |
| stationary minutes  | Count of samples with inference = 0 |
| unknown minutes     | Count of samples with inference = 3 |
| fragmentation       | Number of state transitions         |

Fragmentation counts state transitions per day (e.g., walk → run → stationary).

### How it works

1. Sorts activity samples by timestamp.  
2. Groups by (`participant_id`, `date`).  
3. Counts minutes per inference type.  
4. Tracks previous inference to compute fragmentation.  
5. Outputs one document per day into `per_activity`.

### How to run

```sh
python activity_per.py
```

---

## 3. `conversation_per.py` — Daily Conversation Features

**Source:** `conversation_per.py`

### What it does

Converts raw audio-detected conversation segments into daily social behaviour metrics.

### Methodology

Each row contains:

- `start_timestamp`
- `end_timestamp`

### Extracted attributes

| Attribute              | Description                          |
|------------------------|--------------------------------------|
| conversation duration  | Total speaking minutes per day       |
| episodes               | Number of conversation windows       |

### How it works

1. Sorts segments by start time.  
2. Groups by (`participant_id`, `date`).  
3. Calculates `duration = end - start`.  
4. Counts number of segments.  
5. Saves into `per_conversation`.

### How to run

```sh
python conversation_per.py
```

---

## 4. `bluetooth.py` — Daily Proximity Features

**Source:** `bluetooth.py`

### What it does

Extracts social proximity patterns from Bluetooth scans and stores them in `per_bluetooth`.

### Methodology

Each scan contains:

- `MAC`
- `RSSI` (signal level)
- `time`

### Extracted attributes

| Attribute          | Description                                            |
|--------------------|--------------------------------------------------------|
| unique_devices     | Number of distinct MAC addresses detected              |
| close_devices      | MACs with RSSI ≥ −70 (likely face-to-face)            |
| proximity_episodes | Distinct encounter windows for each device            |

### Proximity episode logic

A new episode starts when:

- Device not seen before today, **or**
- More than 15 minutes have passed since last detection of the same device

### How it works

1. Sorts scans by time.  
2. Tracks last-seen timestamp per MAC.  
3. Detects start of new social encounters.  
4. Saves metrics to `per_bluetooth`.

### How to run

```sh
python bluetooth.py
```

---

## 5. `gps_per.py` — Daily Mobility Features

**Source:** `gps_per.py`

### What it does

Transforms raw GPS logs into daily mobility measures stored in `per_gps`.

### Columns used

From StudentLife GPS:

- `time`
- `provider`
- `network_type`
- `accuracy`
- `latitude`
- `longitude`
- `altitude`
- `bearing`
- `speed`
- `travelstate`

Only the following are used for modelling:

- `time`
- `latitude`
- `longitude`
- `accuracy`

### Valid GPS point criteria

A GPS row is considered valid if:

- `latitude` is not NaN / None  
- `longitude` is not NaN / None  
- `accuracy ≤ 150` meters  

Otherwise → missing point.

### Extracted attributes

| Attribute            | Description                                                                 |
|----------------------|-----------------------------------------------------------------------------|
| daily_distance_km    | Sum of distances between consecutive valid GPS points                       |
| mobility_range_km    | Farthest point from the day’s centroid (mean lat/lon)                       |
| valid_points         | Number of usable GPS readings                                               |
| missing_points       | Invalid GPS readings (NaN / None / accuracy > 150)                          |
| missing_fraction     | `missing_points / (valid_points + missing_points)`                          |

### Haversine distance formula

Used to compute geodesic distance between two lat/lon points:

```text
d = 2R * asin( sqrt( sin²((lat2 - lat1) / 2)
      + cos(lat1) * cos(lat2) * sin²((lon2 - lon1) / 2) ) )
```

### How it works

1. Sort GPS rows by time.  
2. Discard unusable rows (NaN / None or accuracy > 150).  
3. Compute incremental distance via Haversine.  
4. Compute centroid (mean latitude / longitude).  
5. Compute maximum distance from centroid = mobility_range.  
6. Compute `missing_fraction`.  
7. Save daily summary into `per_gps`.

### How to run

```sh
python gps_per.py
```

---

## 6. `sleep_per.py` — Daily Sleep, Night Inactivity, and Charging Routine

**Source:** `sleep_per.py`

### What it does

Computes daily sleep-related behavioural features using StudentLife’s interval-based sensors `dark`, `phonelock`, `phonecharge`, and the burst-style activity inference. Results are stored in the MongoDB collection `per_sleep`.

### Methodology

StudentLife provides interval-based sensors:

- `dark/` — phone in darkness ≥ 1 hour  
- `phonelock/` — phone locked ≥ 1 hour  
- `phonecharge/` — phone plugged in ≥ 1 hour  

and a high-frequency burst sensor:

- `activity/` — second-level activity inference (stationary / walking / running)

None of these sensors provide consistent minute-level samples.

### How we convert this into a per-minute timeline

Each interval `[start, end]` (dark / lock / charge) is expanded into minute timestamps.

Activity samples (every 2–3 seconds during ON cycles) are bucketed into their corresponding minute:

```python
minute_ts = timestamp - (timestamp % 60)
```

If any activity sample inside that minute indicates walking / running, that minute is treated as **active**. Otherwise, it is considered **stationary**, which is required for detecting sleep.

This gives a unified per-minute timeline where the script can answer:

> At minute X, was the phone dark? locked? stationary? charging?

### Extracted attributes

| Attribute          | Description                                               |
|--------------------|-----------------------------------------------------------|
| sleep_onset        | Timestamp marking the start of the longest sleep interval |
| sleep_duration     | Duration (minutes) of the main sleep block                |
| sleep_fragmentation| Number of additional sleep-like blocks beyond the main one|
| night_inactivity   | Minutes between 00:00–06:00 where the phone was inactive  |
| charging_routine   | Fraction of minutes between 21:00–09:00 charging          |

### How sleep is detected

A minute is classified as sleep if:

```text
dark == True
AND phonelock == True
AND stationary (no walking/running detected in that minute)
```

Sleep is searched inside the nightly detection window:

```text
21:00 (previous day) → 12:00 (current day)
```

The longest continuous sequence of sleep-minutes is treated as the main sleep episode.

### Night inactivity

Night inactivity captures “lying in bed”, resting, or being offline — regardless of whether the user is asleep.

Night window:

```text
00:00 → 06:00
```

A minute counts as night inactivity if:

```text
(dark OR locked) AND stationary
```

Reported as a count of minutes.

### Charging routine

Charging routine represents bedtime habit stability.

Night charging window:

```text
21:00 → 09:00
```

Formula:

```text
charging_routine = night_charging_minutes / total_minutes_in_window
```

The output is a fraction (0.0–1.0) indicating how consistently the device was charged overnight.

### How it works (step-by-step)

1. Read interval data from `dark`, `phonelock`, and `phonecharge`.  
2. Convert each interval into minute-level timestamps.  
3. Convert burst-based activity sensor data into minute buckets and mark stationary minutes.  
4. For each day:
   - Construct the sleep window (21:00 previous → 12:00 same day).
   - Identify sleep-minutes using dark + lock + stationary.
   - Extract the longest continuous block → sleep onset & duration.
   - Compute fragmentation (other blocks).
   - Compute night inactivity (00:00–06:00).
   - Compute charging routine (21:00–09:00).
5. Write a clean per-day record into `per_sleep`.

### How to run

```sh
python sleep_per.py
```

---

## 7. `audio_per.py` — Daily Ambient Audio Context

**Source:** `audio_per.py`

### What it does

Aggregates raw audio inference segments into daily-level audio context features and stores them in `per_audio`.

While `conversation_per.py` focuses specifically on conversation episodes, `audio_per.py` captures the broader acoustic environment, such as how much of the day is dominated by speech-like vs. non-speech audio.

### High-level methodology

StudentLife’s audio sensing typically distinguishes between different acoustic states (e.g., silence, voice, noise). At a daily level, we summarise these into features such as:

- Total duration of speech-like segments  
- Total duration of non-speech / background segments  
- Counts or ratios of different audio categories across the day  

Exact feature names depend on the internal implementation, but the intent is to:

> Capture how “alive” or “quiet” the user’s acoustic environment is, beyond explicit conversation detection.

### How it works (conceptual)

1. Read raw audio segments with start / end times and inference labels.  
2. Group by (`participant_id`, `date`).  
3. Aggregate durations and counts for each inference category.  
4. Store per-day audio context into `per_audio`.

### How to run

```sh
python audio_per.py
```

---

## 8. `wifi_per.py` — Daily WiFi / Place-Context Features

**Source:** `wifi_per.py`

### What it does

Extracts WiFi-based context from raw WiFi scan logs and stores daily summaries in `per_wifi`.

WiFi information is a strong proxy for place routines (home vs campus vs other), indoor time, and environmental stability.

### High-level methodology

From WiFi logs (`time`, `SSID`, `BSSID`, `signal level`, connection flags), we derive daily features such as:

- Number of unique WiFi access points seen  
- Indicators of time spent on a small, stable set of APs (e.g., typical of home or campus)  
- Simple proxies of place routine stability (e.g., how often the user jumps between networks)

### How it works (conceptual)

1. Read WiFi scan records and connection information.  
2. Group by (`participant_id`, `date`).  
3. Count unique APs, visits to frequently-used APs, and other simple WiFi-based statistics.  
4. Store one document per day into `per_wifi`.

### How to run

```sh
python wifi_per.py
```

---

## Full Pipeline Execution Guide

### Step 1 — Import raw sensing data

```sh
python etl.py ./path/to/sensing/
```

### Step 2 — Generate per-day behavioural summaries

```sh
python activity_per.py
python conversation_per.py
python bluetooth.py
python gps_per.py
python sleep_per.py
python audio_per.py
python wifi_per.py
```

This produces the following MongoDB collections:

```text
per_activity
per_conversation
per_bluetooth
per_gps
per_sleep
per_audio
per_wifi
```

---

## Our Attributes

| Directory                         | Attribute              | What It Measures (One Line)                                                           | Why We Calculate It (Behavioural Meaning)                                                                 |
|-----------------------------------|------------------------|----------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| `activity/`                       | walking minutes        | Total minutes inferred as walking                                                     | Captures daily physical activity intensity and routine stability.                                         |
|                                   | running minutes        | Total minutes inferred as running                                                     | Reflects higher-intensity movement and energetic behaviour.                                               |
|                                   | stationary minutes     | Total minutes the user remained still                                                 | Identifies low-movement or low-energy days.                                                               |
|                                   | fragmentation          | Number of activity-state switches per day                                             | Indicates restlessness and irregular movement patterns.                                                   |
| `conversation/`                   | conversation duration  | Total minutes of detected conversation segments                                      | Proxy for verbal social engagement and interaction time.                                                 |
|                                   | episodes               | Number of distinct conversation windows                                              | Captures frequency of social interactions and sociability shifts.                                        |
| `bluetooth/`                      | unique_devices         | Count of distinct Bluetooth devices detected                                         | Measures ambient social exposure (how many people were nearby).                                          |
|                                   | close-proximity devices| Devices with RSSI ≥ −70 (physically close)                                           | Indicates face-to-face proximity and potential social contact.                                           |
|                                   | proximity episodes     | Number of separate close-range encounter windows                                     | Captures frequency of close social encounters across the day.                                            |
| `audio/`                          | audio context features | Aggregate durations / counts of different audio states (speech vs non-speech, etc.)  | Captures how “socially alive” or quiet the acoustic environment is, beyond strict conversation segments. |
| `wifi/`                           | WiFi context features  | Unique access points, stability of common networks, simple WiFi-based context stats  | Acts as a proxy for place routines (home / campus / other), indoor time, and changes in environmental structure. |
| `dark/` + `phonelock/` + `phonecharge/` + `activity/` | sleep_onset            | Estimated start time of nightly sleep                                                | Marks the transition from activity to rest; reflects routine stability.                                  |
|                                   | sleep_duration         | Sensor-derived total nightly sleep interval                                          | Reflects rest quantity and day-to-day deviations.                                                        |
|                                   | sleep_fragmentation    | Number of interruptions in inferred sleep                                            | Indicates disrupted or restless sleep patterns.                                                           |
|                                   | night_inactivity       | Total phone inactivity in the night window                                           | Measures night-time restfulness and behavioural regularity.                                              |
|                                   | charging_routine       | Fraction of the night spent charging                                                 | Captures consistent bedtime habits and daily routine structure.                                          |
| `gps/` (if available)             | daily_distance_km      | Total distance traveled using GPS points                                             | Measures large-scale mobility and changes in daily roaming behaviour.                                    |
|                                   | mobility_range_km      | Farthest distance from the day's central location                                   | Indicates whether the user stayed confined or moved across environments.                                 |
|                                   | missing_fraction       | Fraction of daily GPS samples that were invalid or missing                           | Captures time spent indoors, long inactivity periods, device idleness, or low-mobility states.          |

---

## Methodology

### 1. Daily Feature Extraction from Passive Sensing

#### 1.1 Preprocess raw logs

For each participant in both datasets (StudentLife and College Experience):

- Parse raw logs from activity, conversation, Bluetooth, dark/phonelock/phonecharge, and GPS.
- Convert UNIX timestamps into local time and assign each entry to a calendar day.
- Remove invalid sensor readings (e.g., GPS accuracy > 150m, NaN coordinates, empty values).

(In practice, this is implemented by `etl.py` followed by the `*_per.py` scripts described above, including the extended `audio_per.py` and `wifi_per.py`.)

#### 1.2 Aggregate daily behavioural attributes

For every participant-day, compute:

- Activity: walking minutes, running minutes, stationary minutes, activity fragmentation.  
- Conversation: total conversation duration, number of conversation episodes.  
- Bluetooth: unique devices, close-proximity devices, proximity episodes.  
- Sleep & phone inactivity: sleep onset, sleep duration, sleep fragmentation, night inactivity, charging routine.  
- GPS mobility: `daily_distance_km`, `mobility_range_km`, `missing_fraction`.  
- Audio context: daily speech / non-speech context statistics from audio.  
- WiFi context: daily WiFi-based place and routine indicators from WiFi.

This produces a **participant × day × feature** behavioural matrix.

#### 1.3 Align features with PHQ timelines

For each PHQ assessment, extract a fixed behavioural window (e.g., previous 14 days). This forms the feature context used for modelling symptom change.

---

### 2. Within-Person Baselines and Rolling Deviations

#### 2.1 Rolling baselines (per participant)

For each feature and each day, compute a baseline using a trailing window (e.g., previous 14 days):

- Rolling mean (baseline level)  
- Rolling standard deviation (baseline variability)

Exclude the current day from the window.

#### 2.2 Rolling z-score deviation

For each day:

```text
deviation = (value_today - rolling_mean) / rolling_std
```

This gives a within-person deviation value independent of absolute behaviour levels.

#### 2.3 Domain-level deviation scores

Group features into domains (Activity, Sleep, Conversation, Proximity, Mobility, Audio, WiFi).

For each day, compute:

```text
domain_deviation = average absolute deviation across all features in that domain
```

---

### 3. Change-Point Detection (CPD) and Persistence of Deviations

#### 3.1 Identify deviation episodes

A day is marked as deviant if its deviation exceeds a threshold (e.g., 1 or 1.5 SD). Consecutive deviant days form a deviation episode.

Persistence is defined as:

- The number of consecutive deviant days, and  
- The average deviation magnitude within that run.

#### 3.2 Apply CPD to deviation sequences

Run a change-point detection algorithm (e.g., PELT or Bayesian CPD) on each participant’s domain-level deviation series. This identifies structural shifts in behavioural patterns.

#### 3.3 Connect CPD + persistence to PHQ changes

For each PHQ interval:

- Record whether a CP occurs within the behavioural window.  
- Record whether a persistent deviation episode occurs.  
- Compare patterns between PHQ-worsening vs PHQ-stable intervals.

This tests whether persistent deviation and CPD events precede symptom worsening.

#### 3.4 Implementation in this repository

In the current codebase, these ideas are partially operationalised through a family of change-point detection scripts that work on per-day per-participant series, using z-scored values per participant:

- `change_point_sleep.py`  
  - Input: `per_sleep`  
  - Output: `per_sleep_change_points`  
  - Feature example: `sleep_duration`.

- `change_point_mobility.py`  
  - Input: `per_gps`  
  - Output: `per_gps_change_points`  
  - Feature example: `daily_distance_km`.

- `change_point_activity.py`  
  - Input: `per_activity`  
  - Output: `per_activity_change_points`  
  - Feature example: `active_min` (or equivalent activity metric).

- `change_point_social.py`  
  - Inputs: `per_conversation`, `per_bluetooth`  
  - Outputs: `per_conversation_change_points`, `per_bluetooth_change_points`  
  - Features: `conversation_min`, `unique_devices`, `proximity_episodes`.

- `change_point_audio.py`  
  - Input: `per_audio`  
  - Output: `per_audio_change_points`  
  - Features: audio-derived daily context measures.

- `change_point_wifi.py`  
  - Input: `per_wifi`  
  - Output: `per_wifi_change_points`  
  - Features: WiFi-based context measures.

Each script:

1. Connects to MongoDB (StudentLife DB).  
2. Loads the relevant `per_*` collection.  
3. For each participant, builds a daily time series of the chosen feature.  
4. Z-scores the series at the within-person level (mean / std over that participant’s days).  
5. Applies a PELT / binseg-based change-point algorithm (via the `ruptures` library) with hyperparameters:
   - `max_bkps` (maximum number of change points),
   - `min_mean_diff` (minimum absolute shift in mean between segments),
   - Optional minimum segment length before / after.
6. Stores accepted change points in `per_*_change_points` collections, including:
   - `participant_id`, `feature`, `domain`, `method`
   - `change_date`, `index`
   - `mean_before`, `mean_after`, `delta_mean`
   - Segment lengths and indices.

There are also helper scripts:

- `check_cp_counts.py` — prints total counts per `*_change_points` collection.  
- `inspect_sleep_cp.py`, `inspect_mobility_cp.py`, `inspect_activity_cp.py`,  
  `inspect_social_cp.py`, `inspect_audio_cp.py`, `inspect_wifi_cp.py` —  
  summarise per-participant change-point counts and show example documents for sanity checking.

The full rolling-baseline deviation pipeline described above is conceptually compatible with this implementation; the current scripts operate with per-participant z-scores over the whole series and can be extended to true rolling baselines in future iterations.

---

### 4. Cross-Dataset Transfer of Deviation Patterns

#### 4.1 Harmonise features across datasets

Standardise and match feature sets across StudentLife and College Experience.

#### 4.2 Train-test cross-dataset prediction

Train models on one dataset (e.g., College Experience) and test on the other (e.g., StudentLife). Swap roles for symmetry.

The classifier predicts:

```text
Did depressive symptoms worsen between PHQ assessments?
```

This evaluates whether deviation-based behavioural patterns generalise across datasets.

---

### 5. Behavioural Attribution (ANOVA + SHAP)

#### 5.1 ANOVA

Perform domain-level ANOVA to determine how much variance in symptom worsening is explained by each:

- Activity deviation  
- Sleep deviation  
- Phone inactivity patterns  
- Conversation patterns  
- Proximity dynamics  
- Mobility changes  
- Audio and WiFi context deviations  

#### 5.2 SHAP analysis

Compute SHAP values for the final predictive model to:

- Identify the most influential behavioural features  
- Summarise importance at the domain level  
- Provide interpretable explanations of model behaviour  

---

## Summary

This pipeline integrates:

- Daily passive-sensing features  
- Within-person rolling deviations  
- Persistence-aware change-point detection (via the implemented `change_point_*.py` scripts)  
- Cross-dataset evaluation  
- ANOVA and SHAP interpretability  

to understand and model early indicators of depressive symptom worsening using smartphone and wearable signals.
