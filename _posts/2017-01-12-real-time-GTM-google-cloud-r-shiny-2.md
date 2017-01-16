---
layout: post
title: Creating a real-time forecasting dashboard with Google Tag Manager, Google Cloud and R Shiny - Part one
comments: true
---

In part one of this two part series we walk through the steps to stream data from a [Google Tag Manager](https://www.google.com/analytics/tag-manager/) (GTM) implementation into a [Google App Engine](https://cloud.google.com/appengine/) (GAE) web app, which then adds data to a [BigQuery](https://cloud.google.com/bigquery/) table via BigQuery's data streaming capability.  In part two, we then go into how to query that table in realtime from R, make a forecast use [Shiny](https://shiny.rstudio.com/) and use the JavaScript visualisation library [Highcharts](http://jkunst.com/highcharter/) to visualise the data.

The project combines several languages where their advantages lie: Python for its interaction with Google APIs and its quick start creating your own API on App Engine, SQL to query the BigQuery data itself, R for its forecasting libraries and the reactive Shiny framework, and JavaScript for the visualisation and data capture at the Google Tag Manager end.

## Thanks

This project wouldn't have been possible without the help of the excellent work gone beforehand by [Luke Cempre's post on AnalyticsPros](https://www.analyticspros.com/blog/data-science/streaming-prebid-data-google-bigquery/) for streaming data from Google Tag Manager to BigQuery, and [Joshua Kunst](http://jkunst.com/) for his help with the Highcharts JavaScript.

## Data flows

There are two data flows in this project.  The first adds data to BigQuery:

1. GTM collects data into its dataLayer from a web visit.
2. A custom HTML tag in GTM collects the data you want to stream then calls an App Engine URL with its data payload.
3. The app engine URL is sent to a queue to add the data to BigQuery.
4. The data plus a timestamp is put into a BigQuery row.

Then to read the data:

1. The Shiny app calls an App Engine URL.
2. App Engine queries the BigQuery table and returns the most recent rows as JSON.
3. Shiny reads the JSON and puts it into a reactive dataset.
4. A forecast is made with the updated data.
5. The Highcharts visualisation reads the changing dataset, and adds it to the visualisation.

This blog will cover the first, putting data into BigQuery. 



### Reading realtime data from BigQuery

The next functions then let you query the table for your realtime application.  In production this may be separated out into a different app, but for brevity its here in the same application.

We first define some environmental variables in the `app.yaml` setup file, with the dataset and table and a secret code word:

```
#[START env]
env_variables:
  DATASET_ID: tests
  TABLE_ID: realtime_markedmondsonme
  SECRET_SALT: change_this_to_something_unique
#[END env]
```

The first function then queries the BigQuery table we defined in the environmental variables, and turns it into JSON.  By default it will get the last row, or you can pass in the `limit` argument to get more rows, or your own `q` argument with custom SQL to query the table directly:

```
# queries and turns into JSON
def get_data(q, limit = 1):
	datasetId = os.environ['DATASET_ID']
	tableId   = os.environ['TABLE_ID']

	if len(q) > 0:
		query = q % (datasetId, tableId)
	else:
		query = 'SELECT * FROM %s.%s ORDER BY ts DESC LIMIT %s' % (datasetId, tableId, limit)

	bqdata = sync_query(query)

	return json.dumps(bqdata)
	
```

This next class is called when the "get data" URL is requested.  A lot of headers are set to ensure no browser caching is done which we don't want since this is a realtime feed.

For security, we also test via a `hash` parameter to make sure its an authorised request, and decide how much data to return via the `limit` parameter.  

Finally we call the function above and write that out to the URL response.

```
class QueryTable(webapp2.RequestHandler):

	def get(self):

    # no caching
		self.response.headers.add_header("Access-Control-Allow-Origin", "*")
		self.response.headers.add_header("Pragma", "no-cache")
		self.response.headers.add_header("Cache-Control", "no-cache, no-store, must-revalidate, pre-check=0, post-check=0")
		self.response.headers.add_header("Expires", "Thu, 01 Dec 1994 16:00:00")
		self.response.headers.add_header("Content-Type", "application/json")

		q      = cgi.escape(self.request.get("q"))
		myhash = cgi.escape(self.request.get("hash"))
		limit  = cgi.escape(self.request.get("limit"))

		salt = os.environ['SECRET_SALT']
		test = hashlib.sha224(q+salt).hexdigest()

		if(test != myhash):
			logging.debug('Expected hash: {}'.format(test))
			logging.error("Incorrect hash")
			return

		if len(limit) == 0:
			limit = 1

		self.response.out.write(get_data(q, limit))
```

### Full App Engine Script

The full script uploaded is available in the Github repository here:

[main.py](https://github.com/MarkEdmondson1234/ga-bq-stream/blob/master/main.py)

With this script you then need some configuration files for the app and upload it to your Google Project.  A guide on how to deploy this is and more is available from the [Github repository README](https://github.com/MarkEdmondson1234/ga-bq-stream/blob/master/README.md), but once done the app will be available at `https://YOUR-PROJECT-ID.appspot.com` and you will call the `/bq-streamer` and `/bq-get` URLs to send and get data. 




