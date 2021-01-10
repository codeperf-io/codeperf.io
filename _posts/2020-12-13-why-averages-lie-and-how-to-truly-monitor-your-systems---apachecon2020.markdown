---
layout:	post
title:	"Why averages lie and how to truly monitor your systems — ApacheCon2020"
date:	2020-12-13
---

  ![](/img/1*gZU7UgXSOnG7Yz_bPsSvWg.png)#### Why averages lie

We spend most of our time looking at the reported averages of our monitoring systems, completely disregarding the painful truth that the numbers that we look at and present to our bosses, to our business and make decisions based upon, do not represent our user experience.

This simple fact seems to surprise many people. It feels good looking at steady state monitoring charts.

#### ApacheCon session

In the session I presented at[ ApacheCon @Home 2020](https://www.apachecon.com/acah2020/tracks/observability.html) I tried to discuss why is it important to pay attention to the “higher end” of the percentile spectrum in most application monitoring, benchmarking, and tuning environments and how you can make better usage of the open-source tooling we have at our disposal ( giving examples on both OSS HDR and T-Digest Histograms ).

#### Further references

This session it’s supposed to be white/yellow belt and contains further references for documentation, projects and presentations diving deeper in the discussed subjects:

* [how *not* to measure latency — gil tene](https://www.youtube.com/watch?v=lJ8ydIuPFeU)
* [frequency trails: modes and modality — brendan gregg](http://www.brendangregg.com/FrequencyTrails/modes.html)
* [Metrics, Metrics, Everywhere — Coda Hale](https://www.youtube.com/watch?v=czes-oa0yik)
* [lies, damn lies, and metrics — andré arko](https://www.youtube.com/watch?v=pYbgcDfM2Ts)
* [*most* page loads will experience the 99%’lie server response — gil tene](https://latencytipoftheday.blogspot.com/2014/06/latencytipoftheday-most-page-loads.html)
* [if you are not measuring and/or plotting max, what are you hiding (from)? — gil tene](https://latencytipoftheday.blogspot.com/2014/06/latencytipoftheday-if-you-are-not.html)
* [latency heat maps — brendan gregg](http://www.brendangregg.com/HeatMaps/latency.html)
* [visualizing system latency — brendan gregg](https://queue.acm.org/detail.cfm?id=1809426)
* [t-digest — ted dunning](https://github.com/tdunning/t-digest)
* [hdrhistogram: a high dynamic range histogram](http://hdrhistogram.org/)
* [Prometheus Histograms — Past, Present, and Future](https://promcon.io/2019-munich/talks/prometheus-histograms-past-present-and-future/)
  