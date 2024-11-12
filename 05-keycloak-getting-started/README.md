# Keycloak Getting Started
In this section we will bet aking a look at how to setup Keycloak and access its metrics.

## Adding Keycloak
We add Keycloak to our docker-compose file in a similar way to the other applications, but with a few key differences:

```
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.0.5
    command: >
        start --hostname-strict=false --http-port=8080 --http-enabled=true --https-port=-1
    environment:
      - KC_BOOTSTRAP_ADMIN_USERNAME=kit
      - KC_BOOTSTRAP_ADMIN_PASSWORD=Test1234
      - KEYCLOAK_LOGLEVEL=DEBUG
      - DB_VENDOR=h2
      - KC_HEALTH_ENABLED=true #ENABLES HEALTH
      - KC_METRICS_ENABLED=true #ENABLES METRICS
      - KC_HTTP_METRICS_SLOS=50,100,150,200,250,300,350,400,450,500 #CUSTOMIZES METRICS
    networks:
      - monitoring101
    depends_on:
      java-app-a:
         condition: service_healthy
      java-app-b:
         condition: service_healthy
    ports:
      - "8092:8080"
      - "9000:9000" #MAKES METRICS ENDPOINT ACCESSIBLE

```

Keycloak can be started in a development mode using the start-dev command, however the a metrics endpoint cannot be created when using development mode. Because of this, we use the "start" command along with a few arguments which allow us to quickly start Keycloak without setting up HTTPS.

In order to enable the metrics endpoint, we have to:

1) Set the KC_HEALTH_ENABLED and KC_METRICS_ENABLED environment variables to true, which enable health and metrics endpoints respectively.

2) Mount port 8080 *and* port 9000. Port 8080 is the default port for most of Keycloak. The /metrics and /health are however by default located on port 9000.

3) Finally, we have to add the following code to prometheus.yml in order to make prometheus scrape Keycloak's /metrics endpoint:

```
 - job_name: keycloak
   metrics_path: /metrics
   static_configs:
    - targets:
       - keycloak:9000
```

After running docker compose, Keycloak can then be accessed at localhost:8092 and the metrics will be available at localhost:9000/metrics.

## Configuring Keycloak metrics

Keycloak comes with 3 environment variables which can be used to enable additional metrics that are useful for creating histograms. These are all set to false by default. The environment variables are:

1) KC_CACHE_METRICS_HISTOGRAMS_ENABLED. If set to true, buckets for various cache related metrics are enabled. These metrics do not appear to very useful. 

2) KC_HTTP_METRICS_HISTOGRAMS_ENABLED. If set to true, buckets for various HTTP related metrics are enabled. These metrics are not very useful, because the buckets are automatically set to completely random values.

3) KC_HTTP_METRICS_SLOS. If we set this environment variable to e.g. ```KC_HTTP_METRICS_SLOS=50,100,150,200,250,300,350,400,450,500```, something rather interesting happens. Similarly to when we set ```management_metrics_distribution_slo``` in chapter 4, custom buckets are generated for a variety of HTTP-related metrics. This is an example of what one of our metrics might look like after logging into Keycloak 11 times:

```
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.05",} 5.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.1",} 10.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.15",} 10.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.2",} 10.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.25",} 10.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.3",} 11.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.35",} 11.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.4",} 11.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.45",} 11.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="0.5",} 11.0
http_server_requests_seconds_bucket{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",le="+Inf",} 11.0
http_server_requests_seconds_count{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",} 11.0
http_server_requests_seconds_sum{method="POST",outcome="REDIRECTION",status="302",uri="/realms/{realm}/login-actions/authenticate",} 0.776636004
```

These metrics tell us there were 11 successful login attempts which lasted a total of 0.78 seconds. 5 of the login attempts lasted less than 50ms, 5 of the lasted between 50 and 100ms, and 1 of them lasted between 250 and 300ms.

Buckets can be customized to be any value, however all HTTP metrics will share the same bucket values. 

## What do all of these metrics mean?

One should be very careful when interpreting the meaning of each Keycloak metric. For some reason, Keycloak uses very unintuitive metric names.