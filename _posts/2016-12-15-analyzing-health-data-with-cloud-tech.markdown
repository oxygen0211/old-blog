---
layout: post
title: "Analyzing health data using cloud technology"
date: 2016-12-15 20:30:00 +0100
---

For quite some time, I have been tracking multiple aspects of my health using different services. But what to do with the collected data and how to process it so we can gain insights from it? Can we use similar approaches to those we use for analyzing and monitoring cloud systems? Let's try it out.

# Why? - A little background
For almost all of my life, I have been overweight and have been struggling to keep my weight or even decrease it. During a number of approaches to get rid of some pounds, I have established the habit of two sessions of weight training at the gym per week as a stable in my life, have tried out different nutrition patterns and tracking my calorie intake, weight and activity using different health tracking apps.

During the last one and a half years or so I gave up or altered most of the tracking (apart from my weight) and focused mainly on doing weight training sessions that put hard work on all of my main muscles. This did a lot for my muscle mass, fitness and overall wellbeing, but was not effective for loosing body fat which is also a main asset for me. My approach took another turn about three months ago when I had a longer talk with my trainer at the gym about my situation. I got some tips on how to eat better, changed my training pattern from long sessions to HIIT (high intensity interval training) and started tracking my daily calorie intake again to make sure I don't consume more energy than my resting metabolic rate (RMR) will burn. The idea behind this is to force my body to source the energy needed for motion and mental activity from body fat while using training mainly to prevent muscle decrease.

When I started following this pattern, I could almost immediately see my weight and body fat decrease. However, I could as fast see significant swings in my progress. 

This got me my scratching my head quite a bit and one thought struck me: I consider myself to have quite some understanding on how to run cloud-, application- and server systems and how to methodically monitor performance and analyse problems in this area. However, this seems to be not so true for understanding my body and its metabolic processes. Which are the factors that speed up or slow down my fat metabolism with seemingly consistent inputs? Building on top of this, some other thoughts came up: Can we adopt the approaches we use for analyzing logs, events and system performance on health data? I started researching apps and services that help me reveal correlations between different kinds of health data but couldn't find what I was looking for, so I decided to build a prototype to see if this approach is able to give some insight and reveal some patterns.

# Getting the data
Before doing any analysis, we of course need to get some data to work with first. In regard of health data, this means tracking different health aspects to bring the metrics into the digital world. Since tracking health data can be tedious, I try to restrict it to only the data that is helpful for me. In general, I use two different systems to track different health aspects:

## Weight and body composition: Withings
For almost three years now, I have been using a Withings smart body analyzer for tracking my weight and body fat. During this time, the daily weight-in has become part of my morning routine. The smart body analyzer automatically syncs body weight, body fat percentage, heart rate and room air quality over wifi with the companion app on my iPhone and is integrated with [IFTTT](https://ifttt.com) for fun things. For example, it is able to change the color of the light in my bedroom (where it is located) based on the weight it just measured.

![Withings smart body analyzer](/pictures/withings-scale-top.jpg)

## Nutrition: MyFitnessPal
The other big part of traking my fitness tracking is nutrition. I have tried different apps for this and of course each one comes with benefits and tradeoffs. Currently I am using [Under Armour's MyFitnessPal](https://www.myfitnesspal.com/). What sticks out for me with this one is the big food database, the integrated barcode scanner and that goals for every tracked aspect like activity, calorie intake, macro nutrients, vitamins and much more can be set individually and manually, apart from being calculated by basic information like age, weight, height, gender and estimated activity level. If you are getting into tracking your calorie intake vs. your RMR, I recommend you get it calculated based on a body measurement by some professional with professional equipment like a gym, nutrition coach, or doctor. Here in Germany, sometimes even health insurances offer to do this on some public events. Getting your RMR calculated individually is way more accurate than relying on what fitness apps calculating it based on basic info. I have seen mine being set several hundred calories above what I currently use as a reference. This can make or break weight loss when relying on it each day.

## Putting it together: Apple Healthkit
What I found still missing in the services I have looked at is machine readable access to the gathered data. While a few offer APIs, they all were not available by just adding a key to your account or such and need registration or access self sign up and needed some approval or activation by the maintainers of the services (Still waiting for my access to the MyFitnessPal API!). Others don't offer a proprietary API at all. Also, it can cause much effort to support all the APIs we need one by one, so we need another way to get hand on our data. This is where Apple Healthkit comes in handy. Depending on how it is configured, it will gather all data of supported apps (which both Withings and MyFitnessPal are), syncs the seperately tracked aspects to the other apps (e.g. adding weight data to MyFitnessPal in our case) and offers export of all data as an XML file. As a plus, we could also write our own App to get the data pushed into our system as it is added. This is our entrypoint for doing our own analysis.

# Processing the raw data
Now that we have set up our tracking environment and are able to get the data in a processable format, the real fun begins. For this, I have created a [project on GitHub](https://github.com/oxygen0211/healthkit-analyzer) which holds my custom code to bridge the gap between the data generated by healthkit and visualization.

Originally, this was designed to run on AWS Lambda and be triggered by a new export file being uploaded to S3. The Lambda function then would process the included data and push the analysis to [elastic cloud](https://cloud.elastic.co/). However I found myself fiddling around with configurations of the elasticsearch client to get it working with elastic cloud, the lambda handler to get it working properly with Lambda and S3 and making Logs visible in cloud wathch for a while. Since this is not what I intended to spend much time and effort on, I decided to step down the runtime environment for the prototype while still keeping the architecture open for this way of deploying (further work is needed, though). For working on the actual Problem, I went for a local approach that has a similar data and control flow: A Java main function that loads the XML from the working directory (expecting it to be named "export.xml"), processes it and pushes the analysis to a elasticsearch development client I set up on an old notebook using the elastic's [elasticsearch](https://store.docker.com/images/1090e442-627e-4bf2-b29a-555f57a64ecd?tab=description) and [Kibana](https://store.docker.com/images/41105537-6820-4448-908b-4ac7b31be0c2?tab=description) docker images.

Let's take a look at the main concepts this application uses for processing our data.

## Parsing the XML file and grouping the data by day
Healthkit export data can contain all kinds of data that we might or might not find interesting for our purpose. This heavily depends on what Apps you use with Health and what data you are tracking. Due to this, it doesn't make much sense to just ingest the whole Dataset since we can filter out the data we are interested in in while parsing the raw data. For this, we use an XMLStreamReader that iterates over the whole document and define some custom rules on what to skip and which data to include into our data model.

Each entry of the exported document contains three attributes we are interested in: Its value, which is the data we want to work with, its type which tells us what kind of data we have (basically what the health app uses for showing you different data sets) and its creation data, so we can correlate the data of the single types over time. While iterating over the entries, we check what type of data we have and, if we are interested in it, we will add it to our dataset which will be grouped by days. Currently, we use data with the following type attributes:

* "HKQuantityTypeIdentifierBodyMass" - weight
* "HKQuantityTypeIdentifierDietaryProtein" - consumed protein
* "HKQuantityTypeIdentifierDietaryFatTotal" - consumed fat
* "HKQuantityTypeIdentifierDietaryCarbohydrates" - consumed carbohydrates
* "HKQuantityTypeIdentifierDietaryEnergyConsumed" - consumed energy in calories

If we come across one of these entry types, we will store it in a key-value map with its date as the key value to create a timeline for each data type.

The raw data can contain several entries of one type per day (e.g. calorie intake for breakfast, lunch and dinner), but since they don't correlate over time with the data types (most people don't weigh themselves every time they eat something, normally you do that once per day or less) and the human body doesn't react immediately to new inputs, we have to summarize the single entries over a certain amount of time. One day seems to be the correct granularity here. We have to summarize in regard of the meaning of the single data types. For weight, we will just use the last one entered on that day (we might also use an average, minimum or maximum, but for now, the latest one seems sufficient). For nutrition data and calorie intake, we sum it up for each day. For normalizing the date, we keep the information about day of month, month and year from the original data and set hour, minute and second information to a constant, so we can sum it up easily using the normalized date as the key for Java's Map interface.

## Aggregating data into by-day views
We still have a separation by categories as we can see it in the Apple health app. To make it easier to explore correlations between different datasets in the same time slots, we want to pivot our data from multiple time series into a multidimensional one. For this, we introduce an object that is singular per day and holds the data of all tracked data types plus some aggregations that we calculate during its creation. Currently, the only aggregation we calculate is the weight change to the previous day.

## Ingesting the generated data into Elasticsearch
Since we have brought our data into the format we want, we can begin to do more complicated analysis and exploring. Since our main focus is on finding out the correlations between different health aspects and since we would have to gain further domain knowledge on metabolics and weight loss processes for implementing automated analysis, let's stick to the exploratory approach and keep adding automated analysis as an option for further work.

This is the part where we can leverage technology that we know well from cloud operations. We often have to deal with analyzing and finding correlation in multidemsional time series data when running services and applications in large scale. The most known use case here is log analysis. In its structure, our health data is not much different from an application log or performance data, just the meaning of the data differs. One of the most used tools for solving such problems is the [Elastic stack](https://elastic.co). I have made some good experiences with the elastic stack in analyzing logs as well as other data, so this is the obvious choice for me. Of course we could also use a different stack and for example utilize stream- or batch analysis to do different kinds of analysis.

In our case, the elastic stack fits well for storing, indexing and visualizing the data, but not so much for the ingestion part. We could have used a well configured Logstash or one of Elastic's Beats-Komponents to ectract the needed data directly from the export XML file, but this seems more tedious to me than implementing the needed logic in our own application and then ingesting into Elasticsearch using its Transport API. There is a [Java client implementation](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/transport-client.html) to push data to Elasticsearch's indexing mechanism, so we just need to utilize it's interfaces. After having finished the aggregation of the parsed data, our program uses this client to push the generated data to Elasticsearch's index API to store it in Elasticsearch. After this, we have our data ready for further exploring using Kibana, the frontend application of the Elastic stack.

# Data Visualization
As we can read in many blog entries concerning analytics of system behavior, having data in Elasticsearch and visualizing it solely in Kibana's timeline view mot of the time doesn't reveal that much. You need to have visualization that helps you understand correlations and see patterns. Kibana brings good tools for building visualizations and dashboards. The newest of them is [Timelion](https://www.elastic.co/blog/timelion-timeline), which supplies components and a query language visualize time series data stored in Elasticsearch.

The project repo mentioned above includes a JSON file containing a dashboard with a few visualizations comparing one or more of the collected metrics against weight change. The bars signal the single nutrition data records, their colors indicates if there was weight loss (green) or gain (red) on that day.

![Weight loss vs overall calories](/pictures/screen-weight-change-vs-cals.png)
Weight loss vs overall calories

![Weight loss vs consumed carbohydrates](/pictures/screen-weight-change-vs-carbs.png)
Weight loss vs consumed carbohydrates

![Weight loss vs consumed fat](/pictures/screen-weight-change-vs-fat.png)
Weight loss vs consumed fat

![Weight loss vs consumed protein](/pictures/screen-weight-change-vs-protein.png)
Weight loss vs consumed protein

# Conclusion
With a bit of data processing, we can aggregate health data from different Apps for visualizing and analyzing them with tools we know from developing and operating cloud based data processing services. This is just a small proof of concept. We could source the data with a number of other tools to create different results to help us with understanding other aspects of the data we collected. Also, we could include a lot of other data like heart rate, activity or sleep data to analyze different health aspects (e.g. sleep problems).

For my initial question, what the main factors are that speed up or slow down my weight loss, I think the diagrams above draw a pretty clear picture: The main factor seems to be calorie intake, which is no big surprise. While there is a lot of variation in the consumed amount of macro nutrients (carbohydrates, fat and protein) on days when I lost weight, the majority of them matches with a calorie intake of around 2200 to 2400 calories (which fits very well what my RMR calculation told me). There are days with a higher calorie intake which also produced weight loss, but this is likely coupled to higher activity. My personal conclusion is to stick with having a close eye on the amount of calories I consume.

The variation we can see in the diagrams above also leads to another not so revolutionary conclusion: the human body depends on so many factors that there are no absolutely predictable and hard correlations betweenn inputs (such as calorie intake) and outputs (such as weight gain/loss). Through this, inputs do not always produce immediate and forseeable outcomes, so the data we are working on can only give hints to try out modifications on our behavior and can't solely prove any theories.

![Elasticsearch enabled Withings smart body analyzer](/pictures/wifi-scale-es-stickers-top.jpg)

# What happens next?
For me, this was a fun little experiment that I consider finished for now. If there is an urge to do some more health data analysis in the future, this will surely act as a foundation. If you feel like doing some work on this yourself, feel free to use this project as the base. In this case, I enjoy to hear how the project is doing and what you can gain from it.
