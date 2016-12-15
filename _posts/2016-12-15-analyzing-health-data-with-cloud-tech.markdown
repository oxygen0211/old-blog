---
layout: post
title: "Analyzing health data using cloud technology"
date: 2016-12-15 20:30:00 +0100
---

For quite some time, I have been tracking multiple aspects of my health using different services. But what to do with the collected data and how to process it so we can gain insights from it? Can we use similar approaches to those we use for analyzing and monitoring cloud systems? Let's try it out.

# Why? - A little background
For almost all of my life, I have been overweight and have been struggling to keep my weight. During many approaches to turn get rid to loose some weight, I have established two sessions of weight training at the gym per week as a stable in my life, have tried out different nutrition patterns and tracking my calorie intake, weight, activity using different health tracking apps.

During the last one and a half years or so I gave up or altered most of the tracking (apart from my weight) and focused mainly on intense and effective training sessions which did a lot for my muscle, fitness and general wellbeing, but was not effective loosing body fat which is a main asset for me. This changed about two months again when I had a longer talk with my trainer about those topics. I got some tips on how to eat better, changed my training patterns from long sessions to HIIT (high intensity interval training) and started tracking my daily Calorie intake to not get over my resting metabolic rate to bring my body to source the energy for motion and mental activity from body fat.

While following this approach I could almost immediately see my weight and body fat decrease. However, I could see my swings in my progress as fast. This got me my scratching my head quite a bit and one thought struck me really hard: I consider myself to have quite some understanding for cloud-, application- and server systems and how to methodically analyze and monitor them. However, this is not so true for understanding my body and its metabolic processes. Which are the factors that speed up or slow down my fat metabolism with seemingly consistent inputs? Building on top of this, some other thoughts arose. Can we adopt the approaches we use for analyzing logs, events and (cloud) system performance on health data? I searched for apps and services to reveal correlation between different kinds of health data but couldn't find what quite what I was looking for and decided to build a prototype to see if this approach is able to give some insight and reveal some patterns.

# Gaining data
To do any analysis, we of course need data, so let's start with bringing the information we want to source to the digital world. Since tracking health data can be tedious, I only try to restrict it to only the data that is helpful for me. In general, I use two different systems to track different health aspects:

## Weight and body composition: Withings
For almost three years, I have been using a Withings smart body analyzer for tracking my weight and body fat. In the meantime, the daily weight-in has become part of my morning routine. It automatically syncs body weight, body fat percentage, heart rate and room air quality over wifi with the companion app on my iPhone and is integrated with IFTTT.com for fun things. For example, it is able to change the color of the light in my bedroom (where it is located) based on the weight it just measured.

![Withings smart body analyzer](/pictures/withings-scale-top.jpg)

## Nutrition: MyFitnessPal
The other big part of traking my fitness tracking is nutrition. I have tried different apps for this and of course each comes with benefits and trade offs. Currently I am using Under Armour's MyFitnessPal. What sticks out for me with this one is the big food database, the integrated barcode scanner and that the goals for every tracked aspect like activity calorie intake, macro nutrients, vitamins and much more.

## Putting it together: Apple Healthkit
What I found is still missing in all the services I have evaluated is access to the gathered data. While a few offer APIs, they all were not available by just adding a key to your account or such and need registration or access setup by their Ops team (Still waiting for my access to the MyFitnessPal API!), so we need another way to get it. This is where Healthkit comes in handy. Depending on how it is configured, it will gather all data of supported apps (which both Withings and MyFitnessPal are), syncs the seperately tracked aspects to the other apps (e.g. adding weight data to MyyFitnessPal in our case) and offers export of all data as an XML file. This is our entrypoint for doing our own analysis.
