---
layout: post
title: Efficient anti-sampling with the Google Analytics Reporting API
comments: true
---

Avoiding sampling is one of the most common reasons people start using the Google Analytics API.  This blog lays out some pseudo-code to do so in an efficient manner, avoiding too many unnecessary API calls.  The approach is used in the v4 calls for the R package [`googleAnalyticsR`](http://code.markedmondson.me/googleAnalyticsR/v4.html).

## Avoiding the daily walk

The most common approach to mitigate sampling is to break down the API calls into one call per day.  This has some problems:

* **Its inefficient.**  If you have 80% sampling or 10% sampling, you use the same number of API calls.
* **It takes a long time.**  A year long fetch is 365 calls of 5+ seconds that can equate to a 30mins+ wait.
* **It doesn’t always work.** If you have so many sessions its sampled for one day, you will still have sampling, albeit at a lower rate.

## Anti-sampling based on session size

Google Analytics sampling works as [outlined in this Google article](https://support.google.com/analytics/answer/2637192).  The main points are that if your API call covers a date range greater than set session limits, it will return a sampled call.  

The session limits vary according to if you are using Google Analytics 360 and other unknown factors in a sampling algorithm.  Fortunately, this information is available in the API responses via the `samplesReadCounts` and `samplingSpaceSizes` meta data.  See the [v4 API reference](https://developers.google.com/analytics/devguides/reporting/core/v4/rest/v4/reports/batchGet#ReportData) for their definitions.

These values change per API call, so the general strategy is to make two exploratory API calls first to get the sampling information and the number of sessions over the desired date period, then use that information to construct batches of calls over date ranges that are small enough to avoid sampling, but large enough to not waste API calls.

The two exploratory API calls to find the meta data are more than made up for once you have saved calls in the actual data fetch.

## How it works in practice - 80%+ quicker data

Following this approach, I have found a huge improvement in time spent for sampled calls, making it much more useable in say dynamic dashboards where waiting 30 mins for data is not an option.

An example response from the `googleAnalyticsR` library is shown below - for a month's worth of unsampled data  that would have taken 30 API calls via a daily walk, I get the same in 5 (2 to find batch sizes, 3 to get the data).

```r
> library(googleAnalyticsR)
> ga_auth()
> ga_data <- 
    google_analytics_4(id, 
                       date_range = c("2016-01-01",
                                      "2016-02-01"), 
                       metrics = c("sessions",
                                   "bounceRate"), 
                       dimensions = c("date",
                                      "landingPagePath",
                                      "source"), 
                       anti_sample = TRUE)
                                
anti_sample set to TRUE. Mitigating sampling via multiple API calls.
Finding how much sampling in data request...
Data is sampled, based on 54.06% of visits.
Downloaded [10] rows from a total of [76796].
Finding number of sessions for anti-sample calculations...
Downloaded [32] rows from a total of [32].
Calculated [3] batches are needed to download [113316] rows unsampled.
Anti-sample call covering 10 days: 2016-01-01, 2016-01-10
Downloaded [38354] rows from a total of [38354].
Anti-sample call covering 20 days: 2016-01-11, 2016-01-30
Downloaded [68475] rows from a total of [68475].
Anti-sample call covering 2 days: 2016-01-31, 2016-02-01
Downloaded [6487] rows from a total of [6487].
Finished unsampled data request, total rows [113316]
Successfully avoided sampling
```

The time saved gets even greater the longer the time period you request.

## Limitations

As with daily walk anti-sample techniques, user metrics and unique users are linked to the date range you are querying, so this technique will not match the numbers as if you queried over the whole date range.

The sampling session limit is also applied at a web property level, not View for non-GA360 accounts, so its best to use this on a Raw data View, as filters will cause the session calculations be incorrect.

## Example pseudo-code

R code that implements the above is [available here](https://github.com/MarkEdmondson1234/googleAnalyticsR/blob/master/R/anti_sample.R), but the pseudo-code below is intended for you to port to different languages:

```
// Get the unsampled data
test_call = get_ga_api(full_date_range, 
                       metrics = your_metrics, 
                       dimensions = your_dimensions)

// # Read the sample meta data

// read_counts is the number of sessions before sampling starts
// I make it 10% smaller to ensure its small enough as
// it seems a bit flakey
read_counts = meta_data(test_call, "sampledReadCounts")
read_counts = read_counts * 0.9

// space_size is total amount of sessions sampling was used for
space_size = meta_data(test_call, "samplingSpaceSizes")

// dividing by each gives the % of sampling in the API call
samplingPercent = read_counts / space_size

// if there is no sample meta data, its not sampled. We're done!
if(read_counts = NULL or space_size = NULL):
  return(test_call)
  
// ..otherwise its sampled
// # get info for batch size calculations

// I found rowCount returned from a sampled call was not equal 
// to an unsampled call, so I add 20% to rowCount to adjust
rowCount = meta_data(test_call, "rowCount")
rowCount = rowCount * 1.2

// get the number of sessions per day
date_sessions = get_ga_api(full_date_range, 
                           metric = "sessions", 
                           dimensions = "date")

// get the cumulative number of sessions over the year
date_sessions.cumulative = cumsum(date_sessions.sessions)

// modulus divide the cumulative sessions by the 
// sample read_counts.
date_sessions.sample_bucket = date_sessions.cumulative %% 
                                            read_counts

// get the new date ranges per sample_bucket group
new_date_ranges = get_min_and_max_date(date_sessions)

// new_date_ranges should now hold the smaller date ranges 
// for each batched API call

// # call GA api with new date ranges

total = empty_matrix

for i in new_date_ranges:

  if(new_date_ranges[i].start == new_date_ranges[i].end):
    // only one day so split calls into hourly
    // see below for do_hourly() explanation
    batch_call = do_hourly(new_date_range.start, 
                           metrics = your_metrics, 
                           dimensions = your_dimensions)
  else:
    // multi-day batching
    batch_call = get_ga_api(new_date_ranges[i], 
                            metrics = your_metrics, 
                            dimensions = your_dimensions)
                            
  total = total + batch_call
```

### Per hour fetching do_hourly()

The `do_hourly()` function is very similar to the above code for daily, but with a fetch to examine the session distribution per hour.  I only call it when necessary since it is a lot more API calls and its an edge case. 

If you need anti-sampling for sub-hourly, then you should really be looking at using the [BigQuery Google Analytics 360 exports](http://code.markedmondson.me/googleAnalyticsR/big-query.html). :)

