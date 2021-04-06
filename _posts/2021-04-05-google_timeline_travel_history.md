---
layout: post
comments: true
title: How to Summarize your Travel History in under 5 Minutes
description: Use Google Maps location history to get a record of all your international travel quickly and conveniently.
image: /assets/travel_history.jpg
date: 2021-04-05 14:30:00
categories: [DataScience, Coding, Tools, R, Geospatial]
tags: [DataScience, Coding, Tools, R, Geospatial]
excerpt_separator: <!--more-->
---
**How to use your location history to compile a breakdown of all your international travel. Fast, simple, and valuable for immigration purposes or visa applications. We will use the Google Maps takeout feature and a small Python script**

<!--more-->

## Introduction

When applying for my US visa, one of the questions that USCIS had for me was a breakdown of all my international travel in the last ten years. If you're anything like me, that question can cause a pretty big headache. Not because I have anything to hide but because it is challenging to keep track of all the different trips I make. A visa application is a serious business, and it's vital to answer completely and truthfully. So I started digging into old emails, flight loyalty program records, even paper documents. The whole process took me the better part of two hours. I kept thinking there must be a better way to do this. Now I can say with confidence that there is, thanks to Google Maps' Location History feature!


## Data Extraction

### Getting the Files

There are some pre-requisites. You need to have location history enabled and, depending on the time window of travel history that you are interested in, you should disable the auto-delete. Note that auto-delete was recently changed to OPT-OUT! I like having all that data archived and available, so I made sure to disable it (you can do that at https://myactivity.google.com/activitycontrols). I still don't have a full ten years, but I'll take what I can get.

Next, we need to extract the data from Goggle's systems. I wrote a separate post on [how to download location history data programmatically](https://janlauge.github.io/2017/extracting-location-history/). However, here we're looking to download data for a large number of days simultaneously, so the Google Takeout feature is the better choice because we won't end up hitting API request limits.

Go to https://takeout.google.com/, de-select all products (via the button at the top), then re-select "Location History". Leave the file format as `JSON.` Set the file size specifications according to your preferences. You're very unlikely to come up against the GB file limit with just your location history. The export should take about 10 to 15 minutes. When done, you will receive a `.zip` file to your Gmail account that contains folders for each year and files for each month with location history data.


### Getting Coordinates

Many RStats users dislike `JSON`, but it's a ubiquitous and useful format! Fear not, [Thomas Mock](https://twitter.com/thomas_mock) has written an excellent summary on [how to process `JSON` data with R](https://themockup.blog/posts/2020-05-22-parsing-json-in-r-with-jsonlite/). Looking at the location history ` JSON's, it seems there are two sections in each: `placeVisit` and `activitySegment`. This matches what you can see on the timeline in the Google Maps app, where places that you visited are listed as entries, and your movement between places is labeled with inferred activities like "walking", "running", "on a train", or "flying".

Originally I thought I could just use the places listed in `placeVisit` and extract their country from the address field (usually given in the last row). This worked well for most places, the UK and US but got really messy for Japan and Korea. I changed my approach and went for extracting raw latitude and longitude values instead. Digging in a little deeper, I found longitude-latitude values in five places:
* one pair for each entry in `placeVisit`s given with a `startTimestampMs` and `endTimestampMs`
* zero to many pairs for each `placeVisit` in a nested list in `simplifiedRawPath` with a single `timestampMs` each
* one set of `startLocation`, `startTimestampMs`, `endLocation`, `endTimestampMs` for each entry in `activitySegment`
* zero to many pairs with `timestampMs` for each `activitySegment` in a nested list in `simplifiedRawPath`
* zero to many pairs with `timestampMs` for each `activitySegment` in a nested list in `waypointPath`

Note that `simplifiedRawPath` and `waypointPath` aren't present for every entry. Furthermore, the above list might not be complete. For example, I also noticed `parkingEvents`, but since it was mostly empty for me, There could be other entries depending on the types of observation, sensors, features used with Google Maps that I am not aware of and that I missed here. If that is the case, please let me know in the comments below. I'd love to hear from you!

Based on the information above, I wrote a function to extract latitude-longitude data from the takeout `JSON` files. I use `jsonlite`s `flatten()` to make nested lists of consistent schema into data frames and then invoke `transmute()` along with `unnest()` and `pivot_longer()` where needed to standardize the formatting and create rows with `timestamp`, `lng`, `lat` observations (and some metadata). Observations from each part of the `JSON` get combined with `bind_rows()` to an output `df`. I'm only using the places info for now, but the same framework is easily generalizable to activities:

```r
library(tidyverse)
library(jsonlite)
library(lubridate)
library(assert that)

# extract location data from takeout JSON
get_location_from_json <- function(fname, confidence_threshold=75) {

  message(str_glue('now processing {fname}'))
  json <- fromJSON(txt=fname, simplifyVector=TRUE, flatten=FALSE)

  # check JSON sub-lists look okay
  assert_that({
    assertthat::are_equal(length(json[[1]]), 2)
    assertthat::has_name(json[[1]], list('placeVisit', 'activitySegment'))
  })

  # get sub-elements with place and activity data
  df_places <- json %>%
    pluck(1) %>%
    pluck('placeVisit') %>%
    jsonlite::flatten(recursive=F) %>%
    tibble() %>%
    filter(visitConfidence >= confidence_threshold)

  # get location from places
  df <- df_places %>%
    transmute(
      type = 'places',
      time_start = as.numeric(duration.startTimestampMs),
      lat = location.latitudeE7,
      lng = location.longitudeE7,
      time_end = as.numeric(duration.endTimestampMs)) %>%
    # convert to rows with start and end info
    pivot_longer(starts_with('time')) %>%
    drop_na() %>%
    select(type, time=value, lat, lng)

  # get raw place coordinates if there are any
  if ('simplifiedRawPath.points' %in% colnames(df_places) &
      !all(map_lgl(df_places$simplifiedRawPath.points, is.null))) {
    df <- df_places %>%
      unnest(simplifiedRawPath.points) %>%
      transmute(
        type = 'raw',
        time = as.numeric(timestampMs),
        lat = latE7,
        lng = lngE7) %>%
      bind_rows(df)
  }
  return(df)
}
```

Now we just loop over all `JSON` files in our takeout data folder:

```r
library(fs)

# list all files from takeout
fnames <- fs::dir_ls(
  path='data/Semantic Location History',
  type='file',
  recurse=TRUE)

# read files
df = tibble()
for (fname in fnames) {
  df <- get_location_from_json(fname) %>%
    bind_rows(df, .)
}
```


## Processing

We have the raw data, and we're almost ready to have a look at it. However, before we do so, we should change the format of both the timestamps and the coordinates. Timestamps come as UNIX milliseconds. The `lubridate` package makes it easy to convert them to a human-readable date time. Latitude and Longitude values are stored as long integers, hinted at by the original field names `latitudeE7`/`longitudeE7`. Dividing by `1e7` returns the more commonly used Degrees (°) format (decimal places).

```r
df %<>%
  mutate(
    time = as_datetime(time / 1e3),
    lat = lat / 1e7,
    lng = lng / 1e7)
```

Simple points on a 360 x 180 plot would work, but it would be much better to have a polygon map as a frame of reference for our observations. I opted for `rnaturalearth` because it offered a quick and convenient way to get an `sf` country shapefile into R. By the way, if you're not familiar with `sf`: it is a relatively new geospatial R package that provides simple geometry features in a tidyverse compatible form. Check out [this site](https://r-spatial.github.io/sf/) which has tutorials, vignettes, presentations, cheat sheets, and a wiki!

```r
library(rnaturalearth)
library(rnaturalearthdata)
world <- ne_countries(scale = "medium", returnclass = "sf")
```

## Visualization

Now we can finally create a map of my location history:

```r
df %>%
  ggplot() +
  geom_sf(data=world) +
  geom_point(aes(x=lng, y=lat), color='red') +
  theme_map()
```

![First simple map]({{ site.url }}/assets/locationhistory_firstmap.jpeg)

This seems to be 99% correct. I can see all most observations in the UK and the rest of Europe, where I spent most of my time between 2014 and today. I can also see the trips to the US, New Zealand, China, and Japan accurately. In an earlier version of this map, I had lots of in-flight observations from the North Atlantic near Greenland, the Island of Taiwan, and Indonesia. I got rid of these by excluding activity data altogether. There is also an observation in the South Pacific Ocean, and initially, I had no idea where that was coming from.

By manually cross-checking the Maps timeline, I found that it is related to a coastal road stop I did in the very south of New Zealand. The stop got mapped to "South Pacific" as a Google Maps Place with a `visitConfidence` of just `54`. I decided to exclude low confidence visits (threshold of `75`), which solved the issue.


## Country Information

The goal of this project was to get a breakdown of time spent in different countries. I mentioned earlier that I was hoping to be able to just extract that information from the address field in the places data but that it didn't really work. Instead, I chose the slightly more involved route of intersecting coordinates with country shapefiles. We can use the same `sf` object from `rnaturalearth` that was already loaded above. Let's create a breakdown of places visited per country.

```r
# convert to sf points
pnts <- st_as_sf(df, coords = c('lng', 'lat'), crs = st_crs('WGS84'))
# intersect with countries
pnts_in_countries <- st_intersection(pnts, world)

# breakdown of records per country
pnts_in_countries %>%
  as_tibble() %>%
  group_by(sovereignt) %>%
  summarize(n_places=n()) %>%
  arrange(desc(n))

# A tibble: 25 x 2
   iso_a3 n_places
   <chr>     <int>
 1 GBR       11438
 2 USA        1336
 3 DEU         819
 4 NZL         376
 5 HKG         270
 6 FRA         215
 7 ISR         101
 8 JPN          77
 9 GRC          62
10 RUS          61
# ... with 15 more rows
```

25 countries sound about right.


## Border Crossings

Finally, the breakdown with immigration / emmigration dates. I get those using the `lag()` function from `dplyr` and comparing the country of the previous place to the country of the current place as below:

```r
# get immigration / emmigration events
pnts_in_countries %>%
  as_tibble() %>%
  arrange(time) %>%
  transmute(
    date = as_date(time),
    country = iso_a3) %>%
  transmute(
    date_from = lag(date),
    date_to = date,
    country_from = lag(country),
    country_to = country) %>%
  filter(country_from != country_to)

# A tibble: 141 x 4
   date_from  date_to    country_from country_to
   <date>     <date>     <chr>        <chr>     
 1 2014-05-10 2014-05-12 GBR          USA       
 2 2014-05-16 2014-05-18 USA          GBR       
 3 2014-05-29 2014-06-03 GBR          DEU       
 4 2014-06-03 2014-06-04 DEU          GBR       
 5 2014-07-09 2014-07-14 GBR          DEU       
 6 2014-07-14 2014-07-14 DEU          GBR       
 7 2014-12-19 2014-12-20 GBR          USA       
 8 2015-01-08 2015-01-09 USA          DEU       
 9 2015-01-10 2015-01-11 DEU          GBR       
10 2015-03-30 2015-04-02 GBR          DEU       
# ... with 131 more rows
```

Wow, 141 border crossings in total. That's a bit more than what I would have thought. Then again, just driving from London to Cologne gets you three events (GBR -> FRA -> BEL -> GER).


## Conclusion

You can see how this would have taken me ages to do from old emails and flight records, and I would have probably missed some trips!

DISCLAIMER: Use this code at your own risk. I do not guarantee correctness, especially not when applying it to your own data. Please DOUBLE-CHECK MY WORK and let me know if you run into any issues. Some caveats that I am aware of: exact arrival dates can be inaccurate. Google Maps might not register a place on your timeline on the day of departure or right after arrival. An excellent way to check this is to look at records that don't have the same `date_from` and `date_to` value (see the first row in my data). Also, I think that all dates and times are in GMT, and so the day you entered or left a given country may be captured incorrectly in local time. I might return later to work on that further.

Thank you so much for reading! Let me know in the comments below if you found it helpful and what you would like to read about next!
