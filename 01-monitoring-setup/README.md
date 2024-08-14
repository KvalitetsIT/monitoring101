# The Monitoring Setup
The folder monitoring_setup contains a docker-compose file which makes it easy for you to get started using
* Prometheus
* Grafana

We will be using Prometheus to scrape a number of test applications and we will be using the Prometheus UI to learn more about how to query metrics using PromQL. 
The Prometheus UI is a good tool to
* Get an understanding about your application metrics
* developing and debugging PromQL expressions (that you can use to build dashboards later on)

Grafana can be used for building dashboards as well as setting up alarms.

The docker compose setup is self contained and you start it with 'docker compose up' as you would any other compose setup.

When the monitoring setup has been started up, Prometheus UI and Grafana should be available to you on the following urls:
* Prometheus UI: http://localhost:9090/
* Grafana: http://localhost:3000 (credentials: admin/admin)

## Configuration of Prometheus
Configuration of Prometheus is done in the file prometheus/prometheus.yml

It is here that it is configured which endpoints Prometheus should scrape. 

When you start the setup it is configured to scrape Prometheus itself, Grafana, and a number of other endpoints (that are not started yet but don't worry).

You will also make changes to the configuration file later on. 
Remember: If you make changes to the configuration file it is necessary to remove the prometheus container (with docker compose rm) as the container need to be recreated for the configuration changes to take effect.

## Configuration of Grafana
Configuration of Grafana is done in the files grafana/dashboards/dashboard.yaml and grafana/datasources/datasource.yaml

The datasource.yaml file is for configuring the datasource. By default this is set to look for Prometheus on port 9090. It is important to note that all Grafana dashboards need a Prometheus UID, which is used to determine which instance of Prometheus to use in case multiple instances exist. This UID is set to be "prometheusdatasource". Test dashboards created through the Docker setup automatically use this UID, but test dashboards created elsewhere (e.g. taken from the internet) may use a different UID, which will cause errors.

The dashboard.yaml file configures settings for all dashboards: e.g. where and how they're stored.

Dashboards are stored in grafana/dashboards in json format. In order to add new dashboards to the Docker setup, one should:
* Create or update a dashboard through Grafana.
* Click on the "Share" button while viewing the dashboard, then "Export".
* Do NOT check the "Export for sharing for externally" button.
* Press "Save to file" and move the file to the grafana/dashboards directory.

## Validation the setup
When you have started the compose setup and the two containers are running you can go to the Prometheus UI and try to evalute the following PromQL expression: up

The metric 'up' is a special metric that Promethus records when performing a scrape. It is a boolean where 1 indicates a successful scrape and 0 means that the scrape failed. Prometheus adds the label 'instance' showing which target was scraped.

You should get an output similar to this:
```
up{instance="nginx-exporter:9113", job="nginx"} 0
up{instance="java-app-b:8081", job="java-app"} 0
up{instance="java-app-a:8081", job="java-app"} 0
up{instance="localhost:9090", job="prometheus"} 1
up{instance="dotnet-app:8081", job="dotnet-app"} 0
up{instance="grafana:3000", job="grafana"} 1
```

This means that only Prometheus and Grafana was scraped succesfully at this point - which is what we expect.

