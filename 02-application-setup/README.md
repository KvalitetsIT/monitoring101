## The Application Setup
Monitoring is more fun when you look at some actual applications and some actual metrics.

In this setup we will use two applications (KitHugs is written in Java, and KitNugs is a .Net application) to look at some standard ways of exposing application metrics.

The applications are written and maintained by KvalitetsIT:
* https://github.com/KvalitetsIT/kithugs
* https://github.com/KvalitetsIT/kitnugs

The applications were originally made in order to serve as examples of how to be a "good citizen/application" in a Kubernetes cluster. As this also covers the ability to be monitored these two applications will be well suited for our purposes.

Both applications expose a very simple REST API and depends on a database. In order to make the setup as simple a possible both applications use the same database in this setup. This is of course not best practise but a hack just to keep things simple.

As many applications will run in multiple instances in a production setup we have also made the kithugs application run in two instances (named java-app-a and java-app-b). In order to be able to distribute the load between the two instances, the setup also includes the service called frontend. This is just a standard nginx reverse proxy setup that will distribute the incoming calls to the two instances in a round robin fashion. There is a related service in the setup which has the job of exposing metrics regarding the frontend for Prometheus.

The setup can be started using 'docker compose up'. There are dependencies and healthchecks in place so that most problems regarding start up order and so on should be avoided.

When the containers have started and are healthy you can execute the following PromQL: up which should give 

up{instance="nginx-exporter:9113", job="nginx"} 1
up{instance="java-app-b:8081", job="java-app"} 1
up{instance="java-app-a:8081", job="java-app"} 1
up{instance="localhost:9090", job="prometheus"} 1
up{instance="dotnet-app:8081", job="dotnet-app"} 0
up{instance="grafana:3000", job="grafana"} 1