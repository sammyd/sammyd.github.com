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
Unfortunately, TempoDB doesn't yet allow public access to datasets - the
API requires authentication for both read and write. Therefore the first
part of this stage will be to build a proxy for the TempoDB API. We then
use Cubism.js to interface this proxied API.

## TempoDB Proxy

TempoDB provide a selection of API clients - we used the python one to upload
the data points as they are read off the Arduino. Here I'm going to use the
Ruby one - just 'cos.

The following is part of a really simple Sinatra application which will
interface with the TempoDB API.

{% gist 3740067 %} 
