---
layout: post
title: googleAuthR 0.2.0 Released
comments: true
---

[googleAuthR](https://github.com/MarkEdmondson1234/googleAuthR) is now on CRAN version 0.2.0.

This release is the result of using the library myself to create three working Google API libraries, and tweaking the googleAuthR code to better support the process.  As a result all of these libraries are now able to be authorised with one Google OAuth2 login flow:

* [googleAnalyticsR](https://github.com/MarkEdmondson1234/googleAnalyticsR_public)
* [searchConsoleR](https://github.com/MarkEdmondson1234/searchConsoleR)
* [bigQueryR](https://github.com/MarkEdmondson1234/bigQueryR)

## Batching 

This means the libraries above and any other created with `googleAuthR` can take advatage of batching: this uses a Google API feature that means you can send multiple API calls at once.  As the time for big fetches is usually in the wait for API responses, this is a huge time saver for big data fetches.  

For example, it is now implemented within [googleAnalyticsR](https://github.com/MarkEdmondson1234/googleAnalyticsR_public) when walking through data per day to mitigate sampling.  Testing on a two year walk through landing page data, it sped up from ~20 mins to ~2 minutes(!)

![batching_example]({{ site.baseurl }}/images/batch_example.png)

## Service Accounts

There is also now support for [service accounts](https://developers.google.com/identity/protocols/OAuth2ServiceAccount), meaning no OAuth2 flow is needed: a user can just upload or use a JSON file they download from their own Google API console.  As some APIs like Big Query don't have read only scope yet, this also gives greater security options for apps, as a user can give limited access to a app via projects. 

## Plans for the future

I hope the library is of use, it is certainly making my workflow a lot quicker.  I love hearing what people are doing with it.  Next on my planned list is a `GoogleAuthenticateR` library, that works with pure user authentication of Shiny apps via the G+ API. 

I'm also working with offline authentication, meaning apps can work in the background by saving refresh tokens to a database.  Combined with my new [`stripeR`](https://github.com/MarkEdmondson1234/stripeR) this makes paid Shiny apps that work for users when offline a possibility. 

## Full change list

All changes from news.md is listed below:

* Added ability to add your own custom headers to requests via customConfig in `gar_api_generator`
* Add 'localhost' to shiny URL detection.
* Google Service accounts now supported. Authenticate via "Service Account Key" JSON.
* Exposed `gar_shiny_getUrl` and the authentication type (`online/offline`) in `renderLogin`
* `renderLogin` : logout now has option revoke to revoke authentication token
* Added option for `googleAuthR.jsonlite.simplifyVector` for content parsing for compatibility for some APIs
* Batch Google API requests now implemented. See readme or `?gar_batch` and `?gar_batch_walk` for details.
* If data parsing fails, return the raw content so you can test and modify your data parsing function
* Missed Jenny credit now corrected
* Add tip about using `!is.null(access_token())` to detect login state
* Add HTTP backoff for certain errors (#6) from Johann
* Remove possible `NULL` entries from path and pars argument lists
* Reduced some unnecessary message feedback
* moved `with_shiny `environment lookup to within generated function
* added gzip to headers


[![Analytics](https://ga-beacon.appspot.com/UA-73050356-1/115064340704113209584/googleAuthR/v0.2.0)](https://github.com/MarkEdmondson1234/googleAuthR)


