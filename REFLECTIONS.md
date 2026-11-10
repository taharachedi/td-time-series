# Reflections — TD4 Time series: collection, cleaning, smoothing, projection

## 1. How to check the "no real data available" assumption (Stage 0)
One concrete action: read the API documentation and make one test call with
`past_days=90` (in a browser, the URL alone is enough), then look at the first
and last timestamps returned. This takes two minutes and would have shown that
90 days of real history come back in one request. An assumption about data
availability is a claim like any other: it should be tested before it decides
the design.

## 2. My prediction for Le Mans CO2 (Stage 1)
Before running the script I predicted **rising**. The window covers mid-August
to early November: after the summer growing season, plants absorb less CO2,
and from autumn onwards heating and leaf decay release more, so concentrations
in the Northern Hemisphere usually climb towards their late-winter maximum.
Result: the trend slope was **+0.165 ppm/day** (90 days, 2026-08-12 to
2026-11-09), so the prediction was correct.

## 3. Zero gaps: is the gap-detection code unnecessary? (Stage 2)
No. My run reported 0 gaps, but that only proves this source was complete this
one time, not that it is guaranteed to always be complete. Open-Meteo's model
backfills every hour, but the API could change, return nulls during an outage,
or the pipeline could later be pointed at a real sensor feed that does drop
readings. Without the check, a missing day would be silently skipped and the
rolling mean and trend would be computed on shifted data with no warning. The
code costs almost nothing when there are no gaps, and it protects the result
when there are.

## 4. Rolling mean vs exponential smoothing (Stage 3)
My last 5 values: raw [459.5, 470.6, 463.6, 454.3, 451.3], rolling mean
[450.3, 453.5, 455.6, 456.06, 456.6], exponential smoothing [451.78, 457.43,
459.28, 457.79, 455.84]. After the spike at 470.6 ppm, exponential smoothing
went up faster and is already coming down, while the rolling mean is still
rising because the spike stays in its 7-day window. I would prefer the
**rolling mean** for a weekly report on a slow trend, where one unusual day
should not move the indicator much. I would prefer **exponential smoothing**
for monitoring or alerting, where a sudden real change (a pollution episode,
a broken sensor) must show up quickly.

## 5. What happens if the API call fails every retry? (Stage 4)
`fetch_hourly_co2` raises a RuntimeError after 4 attempts, so
`fetch_series.py` stops before writing anything. The workflow step fails, and
the next steps (`analyze_series.py` and the commit) are skipped, because
GitHub Actions stops the job at the first failing step. Nothing is
overwritten: `data/series.csv` and `docs/data/series_analysis.json` keep last
week's content, and the dashboard keeps working but shows **stale** numbers.
The only visible signals are the red run in the Actions tab (and GitHub's
failure e-mail) and the old "Data last analyzed" date on the page.

## 6. How far would I trust my projection? (Stage 5)
About **3 days**. My slope (+0.165 ppm/day) is tiny compared with the
day-to-day noise: raw values moved by up to ±10 ppm in the last week (from
451 to 470 and back). My projection also starts at 450.19 ppm, below both the
last reading (451.3) and the rolling mean (456.6), because the line is fitted
on all 90 days and not on the recent level. So after a few days the weather
(wind, inversions) matters far more than the trend. Beyond about a week, the
seasonal cycle bends the curve and a straight line is no longer a valid model.

## 7. Why did the pipeline still work after changing the location? (Stage 6)
I changed the location from Le Mans to Sydney (-33.8688, 151.2093). Result:
slope **-0.075 ppm/day** (falling) and a last reading of 441.1 ppm, versus
+0.165 ppm/day in Le Mans. This makes sense: it is spring in the Southern
Hemisphere, so vegetation is growing and absorbing CO2. No other code changed,
because the functions in `analyze_series.py` (`clean_series`, `rolling_mean`,
`exponential_smoothing`, `linear_trend_projection`) only work on generic
(date, value) pairs: they never mention the location, the variable name or
its meaning. Only `fetch_series.py` knows where and what is fetched. One
limit: the honesty note text is hardcoded and still talks about the Northern
Hemisphere's seasons, which is wrong for Sydney. Text that interprets the
data is not as generic as the maths.
