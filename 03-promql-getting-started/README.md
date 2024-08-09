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
