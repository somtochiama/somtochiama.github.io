+++
title = "Understanding Native Histograms in Prometheus"
date = 2026-04-06T15:04:15+01:00
images = []
tags = []
categories = ["prometheus", "technical"]
draft = false
+++

## Introduction

Histogram is a metric type in Prometheus used to track the number (and sums) of observations such as  latency and response times. It allows you to answer questions like `how many requests were completed under 100ms`. In version 2.24.0, Prometheus added support for a new type of histograms called native histogram. This new histogram is similar to the original classic histograms but differ in how bucket boundaries are determined, and how the histogram data structure is represented internally.

Unlike classic histograms, which rely on statically configured bucket boundaries, native histograms do not require predefined boundaries. Instead, they use a schema that determines how bucket boundaries are calculated. This approach allows for a potentially infinite number of buckets while remaining memory-efficient because native histograms are sparse, buckets with zero observations consume almost no space. This blog takes a peek at native histograms.

## Bucket boundaries

Histograms works by keeping a count of each observation that is less than or equal to the upper bound of a bucket. Classic histogram are configured by specifying the upper boundary of each bucket. Picking the bucket is left to the user, they pick values that they care about (if you care that requests take less than `10s`, you'll probably have something setup like `[0.1, 0.2, 0.5, 1, 2, 5, 10]`). All requests taking more than 10s falls into the last bucket whether it takes 10secs or 2 minutes. This works fine for most applications, at ten seconds you are already screaming regardless of how far out the actual numbers are, but if you care about the numbers that don't fall into the defined buckets, this might be insufficient.

Native histograms have dynamic resolution and hence can produce more accurate calculations, while still being memory efficient because they are sparse - a bucket with zero observation takes up no space. The bucket boundaries of a native histogram is determined by what prometheus calls a schema. The scehma controls by how quickly bucket widths grow from one bucket to the next. You define a number between `-4` and `8`, and the bucket width of each successive bucket grows exponentially by `(2 ^ (2 ^ (-schema))`. So if you have a bucket with a schema of 0, the bucket with increases exponentially by `2 (2 ^( 2 ^ -0))` = `2`. This means that the upper boundary of each bucket is twice that of the previous bucket.

![class_vs_native](https://hackmd.io/_uploads/SJOYEWGF-x.png)

The higher the schema number the higher the resolution, since the growthnfactor of the bucket width decreases. 

## Sparseness

Although you theorectically have an infinite amount of buckets, native histograms are sparse. We only keep data for buckets that have seen observations in. Prometheus achieves this by keeping additional information about the layouts of the buckets to easily identify filled buckets. The bucket with the upper boundary of 1 is taken as the zeroth buckets, and negative buckets (that'd record negative observations) are to the left of the zeroth bucket and have negative indexes, while positive bucket are to the left and have +ve indexes. The histogram records spans - continous segments of filled buckets. It has both length (to identify how many buckets are in a span) and offset - its distance from the last span.

```
Histogram {
    spans = []Span
    buckets = []f64
}

Span {
    length u64
    offset u64
}
```

For example a native histogram, which has no observations in its zeroth bucket, but has observed counts in its first to third bucket, and then more counts in its sixth and seventh buckets can be represented as shown.
```
Histogram {
    Spans = []{
        Span{offset: 1, length: 3}
        Span{offset: 2, length: 2}
    }
    buckets: [3, 6, 5, 1, 2] 
}
```

![spans](https://hackmd.io/_uploads/SkHnNbfY-g.png)


The first offset of 2 tells us that there are two empty buckets after the first one, then four filled buckets after that, and so forth. The counts of the filled buckets are contained in the buckets field.


## Mergability

Additionally, Prometheus provides an option to limit the number of buckets and when the number of buckets exceeds this number, it would automatically merge buckets of the histogram into one with a smaller schema.

Histograms with different schemas can be merged to produce a histogram with a lower schema. Because the bucket widths would differ by a power of 2 between successive schemas we can merge neighbouring buckets to get a new histogram a smaller schema. Take the example histograms below, we can merge the first two buckets for the first histogram ([1->2] with count 3 and [2->4] with count 6) to get one bucket 1->4 with count of 9, then the two histograms are easily mergable.

![merging_buckets](https://hackmd.io/_uploads/SJVlBWfYbl.png)


## Quantile estimatation

Quantiles in histogram is commonly used to describe latency or response time behaviours in application e.g to answer questions, like what is the latency of 99th percent of my requests, which is important in many applications. For classic histogram, linear interpolation is used to the estimate the nth percentile. It is assumed that the distribution within the buckets are uniform.

For native histograms, it is assumed that observations within a bucket follow a uniform distribution when the bucket is subdivided exponentially. In other words, if a bucket is split into smaller sub-buckets with exponentially spaced boundaries, the samples are assumed to be evenly distributed across those sub-buckets.

![exp](https://hackmd.io/_uploads/Hy2Nr-MY-e.png)

Echoing code comment in [prometheus code](https://github.com/prometheus/prometheus/blob/4321a5573c4400caa7a767aac66fb7a95cf7c0f6/promql/quantile.go#L340):
> For exponential buckets, we interpolate on a logarithmic scale. On a logarithmic scale, 
> the exponential bucket boundaries (for any schema) become linear (every bucket has the same width).

Exponential interpolation is done by first applying linear interpolation for the exponents of the bucket boundaries, then the final value is obtained by raising 2 to the power of the result.

![exp](https://hackmd.io/_uploads/HJGeDbMtbg.png)


```go
// code has been simplified for clarity here
// the actual code has special hang custom buckets, NaN values and zero buckets
for it.Next() { // returns the next bucket with its count, lower and upper bound (in increasing order).
    bucket = it.At()
    if bucket.Count == 0 {
        continue
    }
    count += bucket.Count // we accumulate the total here
    if count >= rank {   // then check if the rank falls within this bucket
        break
    }
}

logLower := math.Log2(math.Abs(bucket.Lower)) // exponent of lower boundary
logUpper := math.Log2(math.Abs(bucket.Upper)) // exponent of upper boundary

rank = count - rank // how many observations falls within the last bucket
fraction := rank / bucket.Count // what fraction of the bucket is that
power := logLower + (logUpper-logLower)*fraction // linear interpolation on the exponennts
return math.Exp2(power), annos // we take a power of two here to get the actual value.
```

You can look the code here [promql/quantile](https://github.com/prometheus/prometheus/blob/main/promql/quantile.go#L350).

I think native histograms can be useful in a lot of cases, it allows use of labels on histograms without blowing up cardinality and finer grained resolution for more accurate measurements. Native histograms can be enabled using the `scrape_native_histograms` config setting but note that this changes the exposition format to protobuf since text is not yet supported.

References:
- [https://docs.google.com/document/d/1VhtB_cGnuO2q_zqEMgtoaLDvJ_kFSXRXoE0Wo74JlSY/edit?tab=t.0](https://docs.google.com/document/d/1VhtB_cGnuO2q_zqEMgtoaLDvJ_kFSXRXoE0Wo74JlSY/edit?tab=t.0)
- [https://prometheus.io/docs/specs/native_histograms](https://prometheus.io/docs/specs/native_histograms)