# PromQL Getting Started
Let's start by looking at some of the metrics exposed by the KitHugs services. As mentioned, we have two instances running (both will be scraped by Prometheus). Let's start by looking at the metrics exposed by KitHugs.
You can see the data collected by Prometheus by accessing the following URLs:
* java-app-a: http://localhost:8081/actuator/prometheus
* java-app-b: http://localhost:9081/actuator/prometheus
If you look at the Promethus configuration file you can see that both of these URLs are scraped by Prometheus as a part of the 'java-app' scrape job.

The format scraped by Prometheus is documented here: https://prometheus.io/docs/concepts/data_model/

Basically each metric consists of:
* A name: What is measured?
* A number of labels: Classification of what is measured
* The actual metric

For example: A metric showing how many bytes are currently allocated in non-heap storage
```
jvm_memory_used_bytes{area="nonheap",id="miscellaneous non-heap storage",} 1.6649136E7
```

Key points:
* Use snakecase for metric names and include the unit (at the end)
* Always use 'baseunits' such as bytes - you can convert to other units in you dashboards later on
*
*

