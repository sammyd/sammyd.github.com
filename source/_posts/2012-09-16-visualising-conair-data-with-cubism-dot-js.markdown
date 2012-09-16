---
layout: post
title: "Visualising ConAir data with Cubism.js"
date: 2012-09-16 22:38
comments: true
categories: [ruby, d3.js, cubism.js, tempodb] 
---

Hot on the tail of being able to record temperature readings from the
Arduino in the office, we can get some charting on the go.

We have been using [TempoDB](http://tempo-db.com/) to store the temperature
data points - it has a great API for querying your dataset, including
rollups, which are able to summarise your data at a resolution of your choice.
Unfortunately, 
