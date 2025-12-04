Perfect! We can wrap the **entire workflow** into a single **flexible function** that:

1. Computes **daily averages**
2. Computes **daily changes (diffs)**
3. Aggregates **weekly** and **monthly** data
4. Computes **forecast** for specified capacity columns up to a given end date
5. Returns **all three dataframes** (daily, weekly, monthly) with optional forecast

Below is a clean, fully functional solution:

---

```python
import pandas as pd
import numpy as np
from statsmodels.tsa.holtwinters import ExponentialSmoothing
from functools import reduce

def process_capacity_data(
    df, 
    capacity_cols, 
    forecast_end=None,  # e.g., '2026-12-31'
    forecast_method='holt_winters',  # or 'linear'
    weekly_agg='mean', 
    monthly_agg='mean'
):
    """
    df: hourly dataframe with 'time_utc' and numeric columns
    capacity_cols: list of columns to forecast
    forecast_end: date string for forecast horizon
    forecast_method: 'holt_winters' or 'linear'
    weekly_agg / monthly_agg: 'mean', 'sum', 'max', or dict of column:method
    """
    
    # --- 1. Prepare daily averages ---
    df['time_utc'] = pd.to_datetime(df['time_utc'])
    cols_to_avg = df.select_dtypes(include=np.number).columns.tolist()
    
    df_daily = df.set_index('time_utc')[cols_to_avg].resample('D').mean().reset_index()
    df_daily['symm_id'] = df['symm_id'].iloc[0]
    
    # --- 2. Compute daily changes for capacity columns ---
    for col in capacity_cols:
        df_daily[col + '_diff'] = df_daily[col].diff()
    
    # --- 3. Weekly aggregation ---
    if isinstance(weekly_agg, str):
        df_weekly = df_daily.set_index('time_utc').resample('W').agg(weekly_agg).reset_index()
    else:
        df_weekly = df_daily.set_index('time_utc').resample('W').agg(weekly_agg).reset_index()
    df_weekly['symm_id'] = df['symm_id'].iloc[0]
    
    # --- 4. Monthly aggregation ---
    if isinstance(monthly_agg, str):
        df_monthly = df_daily.set_index('time_utc').resample('M').agg(monthly_agg).reset_index()
    else:
        df_monthly = df_daily.set_index('time_utc').resample('M').agg(monthly_agg).reset_index()
    df_monthly['symm_id'] = df['symm_id'].iloc[0]
    
    # --- 5. Forecast ---
    if forecast_end is not None:
        forecast_end = pd.to_datetime(forecast_end)
        last_date = df_daily['time_utc'].max()
        forecast_days = (forecast_end - last_date).days
        
        forecast_list = []
        for col in capacity_cols:
            ts = df_daily.set_index('time_utc')[[col]].asfreq('D')
            
            if forecast_method == 'holt_winters':
                model = ExponentialSmoothing(ts[col], trend='add', seasonal=None).fit()
                forecast = model.forecast(forecast_days)
            elif forecast_method == 'linear':
                from sklearn.linear_model import LinearRegression
                X = np.arange(len(ts)).reshape(-1,1)
                y = ts[col].values
                model = LinearRegression().fit(X, y)
                future_X = np.arange(len(ts), len(ts)+forecast_days).reshape(-1,1)
                forecast = model.predict(future_X)
            else:
                raise ValueError("forecast_method must be 'holt_winters' or 'linear'")
            
            df_f = forecast.reset_index()
            df_f.columns = ['time_utc', f'{col}_forecast']
            forecast_list.append(df_f)
        
        # Merge all forecasts
        df_forecast_all = reduce(lambda left, right: pd.merge(left, right, on='time_utc', how='outer'), forecast_list)
        
        # Merge with daily data
        df_daily = pd.concat([df_daily, df_forecast_all], axis=0, ignore_index=True).sort_values('time_utc').reset_index(drop=True)
        
        # Compute forecasted daily changes
        for col in capacity_cols:
            df_daily[col + '_forecast_diff'] = df_daily[f'{col}_forecast'].diff()
    
    return df_daily, df_weekly, df_monthly
```

---

## ✅ Usage Example

```python
capacity_cols = ['effective_used_capacity', 'used_usable_capacity_tb', 'allocated_capacity_tb']

daily, weekly, monthly = process_capacity_data(
    df=df_hourly,
    capacity_cols=capacity_cols,
    forecast_end='2026-12-31',
    forecast_method='holt_winters',
    weekly_agg='mean',
    monthly_agg='mean'
)

# Daily dataframe now has:
# - daily averages
# - daily diffs
# - forecasted capacities up to 2026-12-31
# - forecasted daily changes

# Weekly/monthly dataframes have aggregated metrics
```

---

This function is fully flexible:

* Change `forecast_method` to `'linear'` if you want a simpler linear trend.
* Pass a dictionary to `weekly_agg` or `monthly_agg` to use different aggregation for different columns.
* Daily differences and forecast differences are automatically computed.

---

If you want, I can **add optional rolling averages (7-day/30-day) and percentage change columns** inside the same function — it’s very useful for capacity trend analysis and growth planning.

Do you want me to include that?
