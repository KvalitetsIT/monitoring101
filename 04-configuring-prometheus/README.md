## Configuring Prometheus
Prometheus comes with a bunch of default metrics - in this section we will take a look at how to configure them to be more useful using recording rules.


### Before we start
Our data goes through following process:

1) Our applications calculate and expose metrics to an endpoint
2) Prometheus scrapes these metrics, transforms them and stores them in a database
3) Grafana reads the metrics, transforms them and visualizes them.

At each of these 3 steps, we can modify our metrics to suit our needs.

Modifying metrics in step 1) requires us to modify the codebase of our applications. This is necessary if we want to define metrics that differ significantly from our default metrics. It is however not worth the effort to make changes to our default metrics at this stage.

Modifying metrics in step 2) is what we will cover in this chapter.

Modifying metrics in step 3) is what we do when we write promQL in Grafana as we did in chapter 3.

The main advantage of choosing to modify metrics in step 2) compared to step 3) of the process is that it makes our modified metrics available in Prometheus (and not just Grafana). This allows us to use the metrics for creating Prometheus alerts. Additionally, it can make it slightly easier to modify and create visualizations in Grafana which rely on complicated promQL queries. 

### Configuring Prometheus
We will take a look at the default metric promhttp_metric_handler_requests_total which tells us the total number of http requests to the /metrics endpoint grouped by http code. This metric gives us one value for each individual http code, which can get quite messy:

```
...
promhttp_metric_handler_requests_total{code="500"} 0
promhttp_metric_handler_requests_total{code="501"} 0
promhttp_metric_handler_requests_total{code="502"} 0
promhttp_metric_handler_requests_total{code="503"} 0
...
```

Our goal is to make Prometheus to group these into 5 bins to make the metrics less cluttered:

```
promhttp_metric_handler_requests_total:1xx 0
promhttp_metric_handler_requests_total:2xx 0
promhttp_metric_handler_requests_total:3xx 0
promhttp_metric_handler_requests_total:4xx 0
promhttp_metric_handler_requests_total:5xx 0
```

First, we need to create a recording_rules.yml file, which will tell Prometheus what metrics we want to group together and how. In this file we define a group for our default metric that we wish to configure and specify each of the new metrics we wish to create like so:

```
groups:
  - name: http_requests_grouped
    rules:
      #Group all 1xx status codes
      - record: promhttp_metric_handler_requests_total:1xx #name of the new metric
        expr: sum(promhttp_metric_handler_requests_total{code=~"1.."}) by (job) or vector(0) #promQL code for calculating the metric
    
      #Group all 2xx status codes
      - record: promhttp_metric_handler_requests_total:2xx
        expr: sum(promhttp_metric_handler_requests_total{code=~"2.."}) by (job) or vector(0)

      #Group all 3xx status codes
      - record: promhttp_metric_handler_requests_total:3xx
        expr: sum(promhttp_metric_handler_requests_total{code=~"3.."}) by (job) or vector(0)

      #Group all 4xx status codes
      - record: promhttp_metric_handler_requests_total:4xx
        expr: sum(promhttp_metric_handler_requests_total{code=~"4.."}) by (job) or vector(0)

      #Group all 5xx status codes
      - record: promhttp_metric_handler_requests_total:5xx
        expr: sum(promhttp_metric_handler_requests_total{code=~"5.."}) by (job) or vector(0)
```

A new group should be defined for each default metric we wish to configure.

The ```record``` field specifies the name of a new metric. For example, promhttp_metric_handler_requests_total:1xx is our name for a metric that calculates the total number of HTTP requests with a code starting with "1". The ```expr``` field contains promQL code, which calculates the new metric from our old default metric promhttp_metric_handler_requests_total.

The ```sum(promhttp_metric_handler_requests_total{code=~"1.."}) by (job)``` part of our promQL code calculates the total jumber of HTTP requests starting with a "1". The ```or vector(0)``` part of the code specifies that if no HTTP requests have been made, Prometheus should return a 0 instead of "No data".

The second step is to mount the recording_rules.yml file in our docker-compose.yml file:

```
services:
  prometheus:
    volumes:
      - ./prometheus/recording_rules.yml:/opt/bitnami/prometheus/conf/recording_rules.yml
```

I've chosen to mount it to the same folder as my prometheus.yml file, but anywhere is fine.

Additionally, we have to tell Prometheus that our recording_rules.yml file is a rule file. This is done by adding the location of the recording_rules.yml file in our Docker container to the prometheus.yml like so:

```
rule_files:
  - /opt/bitnami/prometheus/conf/recording_rules.yml
```

The default for metrics defined through recording rules is 60s. Because of this, we want also to specify in the prometheus.yml file that our new metrics should be calculated every 15s. We do this by adding ```evaluation interval: 15s``` to our prometheus.yml file in this location:

```
global:
  scrape_interval: 10s
  evaluation_interval: 15s
```

The end result is that our new metrics are available in Prometheus and Grafana. It is however important to note that they will not show up the endpoints of our applications.
