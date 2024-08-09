# The Monitoring Setup
The folder monitoring_setup contains a docker-compose file which makes it easy for you to get started using
* Prometheus
* Grafana

The setup is self contained and you start it with 'docker compose up' as you would any other compose setup.

The following components are started and should be available to you on the following urls:

* Prometheus http://localhost:9090/
* Grafana http://localhost:3000 (credentials: admin/admin)

## Configuration of Prometheus
Configuration of Prometheus is done in the file prometheus/prometheus.yml

It is here that it is configured which endpoints Prometheus should scrape. 

When you start the setup it is configured to scrape Prometheus itself, Grafana, and a number of other endpoints (that are not started yet but don't worry).

You will also make changes to the configuration file later on. Remember: If you make changes to the configuration file it is necessary to remove the prometheus container (with docker compose rm) as the container need to be recreated for the configuration changes to take effect.

## Validation the setup
When you have started the compose setup and the two containers are running you can go to the Prometheus Query blabla and try to evalute the following PromQL expression: up

TODO : explain up


You should get an output similar to this:
```
up{instance="nginx-exporter:9113", job="nginx"} 0
up{instance="java-app-b:8081", job="java-app"} 0
up{instance="java-app-a:8081", job="java-app"} 0
up{instance="localhost:9090", job="prometheus"} 1
up{instance="dotnet-app:8081", job="dotnet-app"} 0
up{instance="grafana:3000", job="grafana"} 1
```
This means that only Prometheus and Grafana was scraped succesfully which is what we expect at this point.

