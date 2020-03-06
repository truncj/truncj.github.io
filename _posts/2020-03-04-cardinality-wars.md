---
layout: post
title: Cardinality Wars
feature-img: "assets/img/blog/cardinality/lucky_charms.png"
thumbnail: "assets/img/blog/cardinality/lucky_charms.png"
image: "assets/img/blog/cardinality/lucky_charms.png"
---

Deciding on a metrics platform can often be a divisive issue within an organization. Those foundational decisions are often made long before you start on a project. With the increased popularity of cloud native migrations, teams have a unique opportunity to propose a new metrics architecture. The idea of a unified metrics platform across an organization has become a focal point for many clients. A unified platform not only provides lower operational overhead, but enforces a company-wide standard for development teams. 

I’ve spent a lot of time recently architecting production-grade Kubernetes platforms. If you’re familiar with Kubernetes, you’ll know that a large part of that is robust metrics and alerting. We accomplish this with the use of Prometheus, a multi-dimensional time series database for monitoring. It excels in certain scenarios, but we’ll go over why it doesn’t work in all use-cases. 

Prometheus is designed for real-time numeric time series. What that means is that it doesn’t perform well when high cardinality data is involved. Cardinality refers to the number of unique values within a set. For our refresher on cardinality, we can think of a bowl of lucky charms. 

<p align="center">
  <img src="/assets/img/blog/cardinality/lucky_charms.png" width="80%" title="lucky charms">
</p>
<p align="center">
  <i>Image: Twitter/Lucky Charms</i>
</p>

On average, Lucky Charms has a total of 287 marshmallows in a standard box of cereal. For those of us who have memorized the song, there are 8 variations of those marshmallows. We’ll assign them values from 1 to 8. If we were randomly choosing marshmallows out of that bowl and inserting them into Prometheus, it would look like the table below.

<table class="tg">
  <tr>
    <th class="tg-0pky">timestamp</th>
    <th class="tg-0pky">value</th>
  </tr>
  <tr>
    <td class="tg-0pky">12:01 PM</td>
    <td class="tg-0pky">Horseshoe (3)</td>
  </tr>
  <tr>
    <td class="tg-0pky">12:02 PM</td>
    <td class="tg-0pky">Clover</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:03 PM</td>
    <td class="tg-0lax">Star (2)</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:04 PM</td>
    <td class="tg-0lax">Rainbow (7)</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:05 PM</td>
    <td class="tg-0lax">Clover (4)</td>
  </tr>
</table>

Since Lucky Charms only has a total of 8 possible values that will continue to repeat, it would be considered a low cardinality dataset. The data stored within Prometheus would look like this

```shell
lucky_charms.marshmallows = [(12:01, 3), (12:02, 4), (12:03, 2)...]
```

Prometheus will continue to be performant with millions of data points, nevermind the 287 marshmallows we have. If we make a small change, like adding a unique id label, we can substantially lower that performance. 

<table class="tg">
  <tr>
    <th class="tg-0pky">timestamp</th>
    <th class="tg-0lax">pick_id</th>
    <th class="tg-0pky">value</th>
  </tr>
  <tr>
    <td class="tg-0pky">12:01 PM</td>
    <td class="tg-0lax">00001</td>
    <td class="tg-0pky">Horseshoe (3)</td>
  </tr>
  <tr>
    <td class="tg-0pky">12:02 PM</td>
    <td class="tg-0lax">00002</td>
    <td class="tg-0pky">Clover</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:03 PM</td>
    <td class="tg-0lax">00003</td>
    <td class="tg-0lax">Star (2)</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:04 PM</td>
    <td class="tg-0lax">00004</td>
    <td class="tg-0lax">Rainbow (7)</td>
  </tr>
  <tr>
    <td class="tg-0lax">12:05 PM</td>
    <td class="tg-0lax">00005</td>
    <td class="tg-0lax">Clover (4)</td>
  </tr>
</table>

Prometheus will now store the data as follows:

```shell
(name=”lucky_charms.marshmallows”,pick_id=”00001”) = [(12:01, 3)]
(name=”lucky_charms.marshmallows”,pick_id=”00002”) = [(12:01, 4)]
(name=”lucky_charms.marshmallows”,pick_id=”00003”) = [(12:01, 2)]
```

For each unique combination of labels Prometheus and other time series databases will treat them as separate time series datasets on the backend. Each query that involves this data will actually be multiple queries against each unique time series. These queries will continue to be functional for relatively low cardinality data. The downside is that processing time and memory consumption will continue to grow with queries that span an ever increasing number of time series. Eventually, you’ll hit an impasse. 

<p align="center">
  <img src="/assets/img/blog/cardinality/gauge_prom.png" width="80%" title="lucky charms">
</p>
<p align="center">
 <i>Figure 1. Real-life Prometheus time series data with unique label “user”</i>
</p>

Prometheus doesn’t recommend having more than a handful of labels with more than 100 unique values. This starts to be problematic once applications begin to expect per request (or user, host, pod, etc..) metrics on this unified platform. 

In order to combat this issue, we have to start separating out the different types of “metrics” that we’re collecting. Since high and low cardinality data are treated so differently, we can split them out into two different systems. Instead of a metrics platform, this type of high cardinality data is much better suited for what’s traditionally thought of as a logging platform. Utilizing a system like Splunk or ElasticSearch for event streams and logging will allow users to trace individual requests and perform aggregate queries. This allows us to maintain a performant metrics platform along with a separate system for granular data that developers want. 

Large players in the data industry like Splunk have caught on to this need and acquired a company called SignalFX. They offer many of the standard APM features, but one of their main selling points is their ability to handle large amounts of high cardinality data. Behind the scenes SignalFX is splitting up the time series data and the corresponding metadata into two separate systems. Whether you use a proprietary solution or OSS like ElasticSearch, most companies will need to tackle this hurdle at some point. Have you figured out a better or simpler way? Post a comment below about your cardinality battle within your APM stack.

