---
layout: post
comments: true
excerpt_separator: <!--more-->
title:  Database Connections in rMarkdown
date:   2017-09-15 23:00:00
categories: [DataScience, DataBases, R]
tags: [DataScience, DataBases, R]
---
**Connecting R to an enterprise data warehouse? Do it properly and do not hard-code your passwords! Here is how you can do it in R with rMarkdown and RStudio version 1.0+**
<!--more-->

### The Problem
Working as a data scientist in a large organisation, chances are you will have to get data out of an Enterprise Data Warehouse (EDW) and into your Data Manipulation Environment (DME, usually R, Python, Julia, or SAS). Of course, you could create a manual extract, save it as .csv and read it from disk. However, this approach has a number of downsides:
- the manual workflow may be hard to reproduce later on
- files use up additional disk space
- csv files do not store data types

Generally, I prefer to connect my DME directly to the database, as do many other Data Scientists. What I have repeatedly come across in this context is people hard-coding their passwords and access tokens into their analysis code. In my opinion, this is a dangerous practice! It is most likely in violation of the security regulations of your organisation, and for good reason. It is far too easy for your code to accidentally end up on an unrestricted access github repository, an unprotected S3 bucket, or similar. With GDPR just around the corner, a mistake like that could soon cost your organisation up to 3% of their global annual revenue in fines!

### The Solution
So what's the "proper" way to do this? Well, RStudio (v1.0+) offers some great new features in this context. If you are using Windows (like many big corporations do) and you are connecting to your EDW using the Windows ODBC Data Source Administrator, you can read your connection details directly from there using the "odbc" package.

*Note: each code block below should be a chunk in an rMarkdown*
```R
# Unfortunately, odbc is not on CRAN yet.
# So if you do not have it yet we will need devtools
install.packages(devtools)

# Using devtools, we can now install the odbc package
devtools::install_github('rstats-db/odbc')

# Get connection info from Windows ODBC Data Source Administrator
# Using the name you set manually
con <- dbConnect(odbc::odbc(), 'EDW_name')
```

Using this connection object, we can now write **and run** SQL code snippets in rMarkdowns, rNotebooks, and shiny apps. Just pass the connection as property to the snippet and specify an "output.var" that will capture the output. This "output.var" will be available in your R workspace afterwards.

```sql
-- This should be a chunk with the following header:
-- {SQL, connection = con, output.var = result}
-- As a result, this turns into sql code.
-- comments need to be marked accordingly
SELECT TOP 10 * FROM EDW_database.EDW_table
```

```r
# result will be available in the next chunk!
result
```

This code has syntax highlighting, runs start to finish without any manual steps, does not rely on "hacky" string queries, does not have hard-coded passwords, and your data updates as and when new data becomes available in your EDW!

As always, hope this is useful for someone. Please leave a comment below!
