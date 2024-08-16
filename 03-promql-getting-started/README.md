# PromQL Getting Started
Let's start by looking at some of the metrics exposed by the KitHugs services. As mentioned, we have two instances running (both will be scraped by Prometheus). Let's start by looking at the metrics exposed by KitHugs.
You can see the data collected by Prometheus by accessing the following URLs:
* java-app-a: http://localhost:8081/actuator/prometheus
* java-app-b: http://localhost:9081/actuator/prometheus

If you look at the Promethus configuration file you can see that both of these URLs are scraped by Prometheus as a part of the 'java-app' scrape job.

## What Metrics Do We Have and How Do They Look?
The format scraped by Prometheus is documented here: https://prometheus.io/docs/concepts/data_model/

Basically each metric consists of:
* A name: What is measured?
* A number of labels: Unique identification of what is measured in terms of key-value pairs associated with the time series.
  * Instrumentation labels: Labels known within you application (for example: The path of a timed HTTP request)
  * Target labels: Related to architecture and deployment (for example: The id of the Kubernetes worker currently running the service scraped) 
* The actual metric: A number

For example: A metric showing how many bytes are currently allocated in non-heap storage in our KitHugs application.
```
jvm_memory_used_bytes{area="nonheap",id="miscellaneous non-heap storage",} 1.9377264E7
```

There are [different kinds of metrics](https://prometheus.io/docs/concepts/metric_types/).

The example above is a gauge metric. This can be concluded from the metadata in the Prometheus output (see the #TYPE annotation). Notice the instrumentation labels describing the memory area and id.

A gauge metric can increase and decrease.

Key points when defining metrics:
* Use snakecase for metric names and include the unit (at the end)
* Always use 'base units' such as bytes, seconds - you can convert to other units in frontend tools for example Grafana dashboards.
*
*

## How Can We Query Metrics in Prometheus?
After Prometheus scrapes the various targets, the results will be stored in the Prometheus time series database. In order to query the metrics we will use the [Expression browser in the Prometheus UI](http://localhost:9090).

Try and query for jvm_memory_used_bytes.
You should get a result with a number of metrics matching the prometheus endpoint output for the java-app-a and java-app-b. 

```
...
jvm_memory_used_bytes{area="nonheap", id="miscellaneous non-heap storage", instance="java-app-b:8081", job="java-app"} 19377264
...
```

Notice how Prometheus has added target labels for job and instance.

## Aggregation on Labels
As you can see there are individual metrics for heap and nonheap memory as well as the two instances currently scraped. If we instead want to the total memory consumption for "heap" and "nonheap" we can use the sum [aggregation operator](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators) with the 'without' clause naming the labels we want to ignore:

Try to evaluate: sum without (instance, id) (jvm_memory_used_bytes)

Alternatively, we can specify the labels to keep:

Try to evaluate: sum by (area, job) (jvm_memory_used_bytes)

Question: What happens if a label (either target or instrumentation labels) is added? How will it affect the output in the two cases above?

Answer: The 'by' removes labels that it does not know about whereas 'without' does not. In that sense 'without' is probably "safer" unless you know exactly what you are looking for (for example in the case of "Info" todo link).


## Info Labels

sum by (job, version) (up * on (instance, job) group_left(version) application_information)

## Visualizing counters
### Overview

In this section we will take a look at using the increase and rate functions for visualizing counter metrics. We'll do this by taking a look at the prometheus_http_requests_total metric, which for the purpose of this section was set to only visualize the HTTP requests to the /metrics endpoint. 

The most basic way of visualizing data is to simply visualize the counter in a time series:
```
...
prometheus_http_requests_total
...
```
![screenshot](images/increase_rate_1.png)

This produces a graph which shows a clear increasing trend in the number of HTTP requests. However, it is difficult from this visualization to assess the rate at which the number of HTTP requests is increasing. Additionally, it is difficult to assess whether the rate at which HTTP requests are being made is constant, increasing or decreasing over time.

There are 2 ways of dealing with this.

### Increase
The first method is the increase function which takes a unit of time as an argument (in this case 20s): 
```
...
increase(prometheus_http_requests_total[20s])
...
```
![screenshot](images/increase_rate_2.png)

The increase function in this example is set to a range of 20 seconds. This produces a visualization where every point on the y-axis shows the number of HTTP requests that were produced in the 20 seconds leading up to the time displayed on hovering on the point. 

For example, we see two spikes at the times 10:08:30 and 10:12:00. These spikes hit 20 and 22 on the y-axis respectively. This means that 20 HTTP requests were made between 10:08:10 and 10:08:30, and 22 HTTP requests were made between 10:11:40 and 10:12:00.

It is important to note that the points on the graph overlap in time. For example, we see a point at the timestamp 10:11:45. This point shows us that between 10:11:25 and 10:11:45 2 HTTP requests happened - note that the end time of this interval is after the start time of the next interval. 

### Rate
Now what happens if we replace increase by rate?:
```
...
rate(prometheus_http_requests_total[20s])
...
```
![screenshot](images/increase_rate_3.png)

The graph looks exactly the same. The rate and increase functions always produce graphs that look exactly the same. The only change is on the y-axis.

The rate function works the same as the increase function - it just shows the increase per second during the interval of 20 seconds, instead of the total increase during the interval.

When hovering over the spikes, we are now given the values of 1.00 and 1.10 respectively. This means that HTTP requests were occurring at a rate of respectively 1.00 and 1.10 HTTP requests per second during the two spikes.

### Comparison
So when should we use rate, and when should we use increase?

Using rate gives us an estimate of the increase in requests per second. This makes sense when we are visualizing time intervals that are so small that it makes sense to use seconds as a unit. An example of this would be if we wanted only wanted to visualize the HTTP requests that were made in the past 5 minutes.

When we use increase, we can (and should) interpret the points as estimates of the increase per [time unit]. For example, 

```
...
rate(prometheus_http_requests_total[1h])
...
```
gives us an estimate of the hourly increase in HTTP requests. This makes sense for data where we do not care about small rapid changes in data, and are more interested in longer trends.

### Interval length
The longer we choose our intervals to be, the more they will overlap. This makes for prettier graphs that do not display rapid changes but instead make it easier to see long-term trends in data. Here we try setting the interval length to 10 minutes:
```
...
rate(prometheus_http_requests_total[10m])
...
```
![screenshot](images/increase_rate_4.png)

There is no objective answer to how long the intervals should be but a good rule of thumb is to make the interval lengths roughly 1/100th of the total width of the graph we're making. Additionally, it is good practice to set the interval length to be either 1 second, 1 minute, 1 hour or 1 day as this is much less likely to lead to misinterpretations of the data. For example, when making a graph visualizing the HTTP requests made over the past 3-7 days, it would make sense to use the increase function and choose the interval lengths to be 1 hour:
```
...
rate(prometheus_http_requests_total[1h])
...
```
## Visualizing gauges
### Intro to deriv functions
We've covered the increase and rate functions, which are used for counters. Gauges are another type of Prometheus metric, which is used for data that can both increase and decrease over time. Prometheus provides a very neat function for dealing with gauges, which is called deriv. Unfortunately deriv does not work with counters, only gauges.

Let's start by visualizing the number of heap bytes in use by Grafana:
```
...
go_memstats_heap_inuse_bytes{job="grafana"}
...
```
![screenshot](images/deriv_1.png)

This provides us with a nice graph, but in some cases it can be difficult to judge whether our data is increasing or decreasing over time. One way of dealing with this is to use the deriv function:
```
...
delta(go_memstats_heap_inuse_bytes{job="grafana"}[1m])
...
```
![screenshot](images/deriv_2.png)

The deriv function works somewhat similarly to the rate function and provides us with an estimate of the change in bytes per second over the period of the time we specifiy in [brackets]. For example, we can see a big spike at the timestamp 11:28:15 with a value of 28,320,250, which tells us that between 11:27:15 and 11:28:15 the number of heap bytes in use was increasing by roughly 28.3MB per second during that timeframe.

### How does deriv work?
The deriv function works fundamentally differently from the rate and increase functions.

In short, data many times every single second. Prometheus provides us with snapshot of this data every, say, 15 seconds. From these snapshots we want to estimate at what rate our data changing.

When we choose to use the deriv function, Prometheus uses linear regression to calculate the best possible estimate of the rate at which our data changes. Without getting into the heavy math as to why, linear regression is by far the best method for estimating the rate of change in data. Because of this, it is not good practice to use alternative functions like delta for gauges.

Additionally, choosing our interval to be rather small (e.g. [1m] or [5m]) will usually give us the best estimates, but it's often perfectly reasonable to set our interval lengths to something longer (e.g. [1h]) if we want a prettier, smoother graph with fewer spikes.

### Interpreting and modifying deriv
Deriv calculates the *change per second*. But what if we want our visualization to display e.g. the hourly change in byte usage? We should take the following 3 steps:

1) We simply multiply the output of the deriv function by 3600 to convert from seconds to hours:

```
...
delta(go_memstats_heap_inuse_bytes{job="grafana"}[1m]) * 3600
...
```

2) We clearly state in the title of the graph that it's a visualization of the *hourly* changes in byte usage.

3) We optionally adjust the interval length to be [1h] or something similar. This is completely optional but should generally provide a good compromise an accurate and nice-looking graph.

Similarly, one could e.g. visualize the per-minute changes by multiplying the output of the deriv function by 60, or one could divide the output of the deriv function by 1,000,000 to get the changes per second measured in MB.