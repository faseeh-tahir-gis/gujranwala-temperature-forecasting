# Methodology

## 1. Data collection

Data are requested from the Open-Meteo Archive API (`https://archive-api.open-meteo.com/v1/archive`) for latitude 32.1877 and longitude 74.1945, in the Asia/Karachi timezone.

| Variable | Open-Meteo name | Column |
|---|---|---|
| Air temperature at 2 m | `temperature_2m` | `temperature_C` |
| Precipitation | `precipitation` | `precipitation_mm` |
| Relative humidity at 2 m | `relative_humidity_2m` | `humidity_pct` |

Two windows are downloaded, ending two days before the run date because the archive lags real time:

| Window | Length | Use |
|---|---|---|
| Short | 7 days (168 hourly records) | Twice-daily observation log |
| Extended | About 120 days (2,904 hourly records) | Model training and testing |

## 2. Observation log

From the 7-day hourly data, only the 08:00 and 20:00 readings are kept for each day, giving 14 readings with temperature, rainfall and humidity. The table is saved as `data/gujranwala_7day_observations.csv`.

## 3. Daily aggregation

Hourly values are grouped by date:

| Daily field | Rule |
|---|---|
| `Tmax` | Maximum hourly temperature |
| `Tmin` | Minimum hourly temperature |
| `Tmean` | Mean hourly temperature |
| `Rainfall_mm` | Sum of hourly precipitation |
| `Humidity_mean` | Mean hourly relative humidity |

Rows with missing values are dropped. The run documented here had none, leaving 121 daily records.

## 4. Feature engineering

The target is the next day's `Tmax` (`Tmax_next_day`, made with `shift(-1)`).

| Feature | Definition |
|---|---|
| `Tmax_lag1`, `Tmin_lag1`, `Tmean_lag1`, `Rainfall_lag1`, `Humidity_lag1` | The current row's day values (the latest day known when predicting the next day) |
| `Tmax_roll3`, `Tmax_roll7` | Rolling mean of `Tmax` over 3 and 7 days |
| `Rainfall_roll3`, `Rainfall_roll7` | Rolling sum of rainfall over 3 and 7 days |
| `Rained_yesterday` | 1 if the day's rainfall exceeds 0.1 mm, otherwise 0 |
| `day_of_year` | Calendar day number, 1 to 366 |

Rows with any missing value (the first 6 for the 7-day rolling window, and the last because it has no next-day target) are dropped, leaving 114 rows.

## 5. Train/test split

The data are split in time order, without shuffling:

| Set | Rows | Dates |
|---|---|---|
| Train | 91 | 29 June to 27 September 2026 |
| Test | 23 | 28 September to 20 October 2026 |

## 6. Models

### XGBoost

| Setting | Value |
|---|---|
| `n_estimators` | 300 |
| `max_depth` | 4 |
| `learning_rate` | 0.05 |
| `subsample` | 0.8 |
| `colsample_bytree` | 0.8 |
| `random_state` | 42 |

### LSTM

| Setting | Value |
|---|---|
| Input | Sequences of 5 days by 11 features, scaled to 0 to 1 with MinMaxScaler |
| Layers | LSTM(64, return sequences), Dropout(0.2), LSTM(32), Dropout(0.2), Dense(16, ReLU), Dense(1) |
| Optimizer and loss | Adam, mean squared error |
| Training | Up to 150 epochs, batch size 8, 15% validation split |
| Early stopping | Patience 15 on validation loss, best weights restored (stopped after 54 epochs in the documented run) |
| Sequences | 109 total, 87 for training and 22 for testing |

Predictions are converted back to degrees Celsius with the inverse of the target scaler.

## 7. Evaluation

| Metric | Formula |
|---|---|
| MAE | Mean of absolute errors |
| RMSE | Square root of the mean squared error |

Both are in degrees Celsius and computed on the test period only.

## 8. Reproducibility

Random seeds are set for NumPy and TensorFlow (42) and XGBoost (`random_state=42`). Because the notebook downloads new data on every run, the training window moves forward each day and results will not match the figures in this repository exactly.
