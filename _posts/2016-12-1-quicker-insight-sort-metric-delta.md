---
layout: post
title: Insights sorting by delta metrics in the Google Analytics API v4
comments: true
---

As analysts, we are often called upon to see how website metrics have improved or declined over time.  This is easy enough when looking at trends, but if you are looking to break down over other dimensions, it can involve a lot of ETL to get to what you need.

For instance, if you are looking at landing page performance of SEO traffic you can sort by the top performers, but not by the top *most improved* performers.  To see that you need to first extract your metrics for one month, extract it again for the comparison month, join the datasets on the page dimension and then create and sort by a delta metric.  For large websites, you can be exporting millions of URLs just so you can see say the top 20 most improved. 

This comes from the fact the Google Analytics web UI and Data Studio don't let you sort by the *change* of a metric.  However, this is available in the Google Analytics API v4 so a small demo on how to it and how it can be useful is shown here.

## Extracting the data

In v4, you can pass in two date ranges in one call.  When you do this a new ordering type comes available, the [`DELTA`](https://developers.google.com/analytics/devguides/reporting/core/v4/basics#delta_ordering) which is what we can use to sort the results.

Bear in mind any metric filters you add will apply to the first date range, not the second.

## Code

The below is implemented in R using [`googleAnalyticsR`](http://code.markedmondson.me/googleAnalyticsR/)

We first load the library, authenticate and set our ViewID:

```r
library(googleAnalyticsR)
ga_auth()

al <- google_analytics_account_list()

gaid <- yourViewID
```

These are some helper functions to get the start and end dates of last month, and the same month the year before:

```r
#' Start of the month
#' @param x A date
som <- function(x) {
  as.Date(format(x, "%Y-%m-01"))
}

#' End of the month
#' @param x A date
eom <- function(x) {
  som(som(x) + 35) - 1
}

#' Start and end of month
get_start_end_month <- function(x = Sys.Date()){
  c(som(som(x) - 1), som(x) - 1)
}

last_month <- get_start_end_month()
year_before <- get_start_end_month(Sys.Date() - 365)
```

We now create an SEO filter as we only want to examine SEO traffic, and a transactions over 0 metric filter:

```r
## only organic traffic
seo_filter <- filter_clause_ga4(list(dim_filter("medium", 
                                                 "EXACT", 
                                                 "organic")
                               ))
                               
## met filters are on the first date
transaction0 <- filter_clause_ga4(list(met_filter("transactions", 
                                                  "GREATER_THAN", 
                                                  0)))
```

This is the sorting parameter, that we specify to be by the biggest change in transactions from last year at the top of the results:

```r
## order by the delta change of year_before - last_month
delta_trans <- order_type("transactions","DESCENDING", "DELTA")
```

We now make the Google Analytics API v4 call:

```r
gadata <- google_analytics_4(gaid,
                             date_range = c(year_before, last_month),
                             metrics = c("visits","transactions","transactionRevenue"),
                             dimensions = c("landingPagePath"),
                             dim_filters = seo_filter,
                             met_filters = transaction0,
                             order = delta_trans,
                             max = 20)
```

You should now have the top 20 most declined landing pages from last year measured by e-commerce transactions.  Much easier than downloading all pages and doing the delta calculations yourself.

If you want to get the absolute number of declined transactions, you can add the column via:

```r
gadata$transactions.delta <- gadata$transactions.d2 - gadata$transactions.d1
```

## Summary

With this data you can now focus on making SEO improvements to those pages so they can reclaim their past glory, at the very least its a good starting point for investigations. 



