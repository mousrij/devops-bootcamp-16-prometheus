## Notes on the videos for Module 16 "Monitoring with Prometheus"
<br />

<details>
<summary>Video: 1 - Introduction to Monitoring with Prometheus</summary>
<br />

Prometheus is an open-source monitoring system and alerting toolkit. It is used widely and has an active community. Prometheus gathers, organizes, and stores metrics as time series data from targets by "scraping" metrics HTTP endpoints. It can also trigger alerts when specified conditions are observed.

Prometheus was designed for monitoring highly dynamic container environments like Kubernetes or Docker Swarm, but it can also be used to monitor traditional bare server applications.

### Why should you use Prometheus?
Monitoring highly dynamic container environments can be very challenging, so it helps to have tools like Prometheus, which are designed for monitoring these types of environments. When you have 100s or 1000s of containers, plus components on multiple levels (infrastructure, platform, application) you need a way to have a visibility and consistent monitoring across all these components. Without visibility, it's a black box for you. When things break inside your complex environment, you have no idea what is happening. You don't know what has caused the issue, what is not working.

**Example:** Many application errors appear in frontend to the end user. They only see the error message, but the cause can be any of the many components in the backend. Monitoring can help identifying the problem quickly with little effort. Instead of manually trying to troubleshoot across multiple components, it will help exactly pin point directly to the root cause.

With Prometheus everything is automated. It's constantly monitoring and looking out for any issues real time and may even identify a potential issue before it happens, so you can prevent it.

Prometheus
- constantly monitors all the services
- triggers alerts when a services crashes
- helps to identify problems before they happen

### Architecture
The `Prometheus Server` is the main component that does the actual monitoring work. It consists of a `Data Retrieval Worker` which scrapes the metrics data from applications, servers or services and stores it in a `Time Series Database`. It also contains an `HTTP Server` which accepts PromQL queries and is used by other components like the `Prometheus Web UI` or `Grafana`.

The applications, servers or services that are monitored by Prometheus are called `Targets`. And the units of these targets that are monitored (like CPU status, memory usage, disk space, requests count, exceptions count, request duration, etc.) are called `Metrics`.

The metrics are stored in human readable format. The contain a `Type` and a `Help` attribute. The type is one of the following:
- Counter: how many times x happened
- Gauge: what is the current value of x now
- Histogramm: how long or how big
The help attribute is just a description of what the metric is.

Prometheus collects the metrics data from targets by pulling that data from HTTP endpoints (`[host-address]/metrics`] the targets expose. The result returned by these endpoints must be in a format that Prometheus understands. Some services expose `/metrics` endpoints by default, others need another component for that, so called `Exporters`. Exporters help in exporting existing metrics from third-party systems as Prometheus metrics. An exporter is a service that fetches metrics from a target and converts the data and exposes them as Prometheus metrics. Prometheus can then scrape this endpoint as usual. Some exporters are maintained as part of the official Prometheus organization, others are externally contributed and maintained.

For example if you want to monitor a Linux server, you
- download a node exporter
- untar and execute it
- it converts metrics of the server
- and exposes a /metrics endpoint
- you configure Prometheus to scrape this endpoint

Exporters are often available as Docker images. If you want to monitor a MySQL database running in a pod inside a Kubernetes cluster, you can add a mysql exporter as a sidecar to that pod. Prometheus also provides monitoring of K8s cluster node resources out-of-the-box.

To monitor your own application, Prometheus provides client libraries for many different languages, that allow you to define and expose internal metrics via an HTTP endpoint on your application's instance.

If the number of targets to be monitored increases and one Prometheus server instance is no longer sufficient for scraping all the metrics endpoints, you can start additional Prometheus servers. These servers can then build a Prometheus federation wherein a Prometheus server can scrape data from other Prometheus servers.

When a target only runs for a short time, shorter than the scraping interval (e.g. a short-living batch job), it is also possible to define a `Push Gateway` the short-living target pushes its metrics to. Prometheus then collects the metrics from this gateway at a later time.

### Configuring Prometheus
You write your configuration in a `prometheus.yml` file to let Prometheus know what to scrape (which targets) and when (at what interval). Targets are discovered via a service discovery mechanism.

Example Config File:
```yaml
# how often Prometheus will scrape its targets
global:
  scrape_interval: 15s
  evaluation_interval: 15s

# rules for aggregating metric values or creating alerts when conditions are met
rule_files:
  - first.rules
  - second.rules

# what resources Prometheus monitors
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']  # Prometheus has its own /metrics endpoint
  - job_name: node_exporter
    scrape_interval: 1m
    scrape_timeout: 1m
    static_configs:
      - targets: ['localhost:9100']
```

Default values for each job are:
```yaml
metrics_path: "/metrics"
scheme: "http"
```

### Alertmanager
The Alertmanager handles alerts sent by Prometheus server. It takes care of deduplicating, grouping and routing them to the correct receiver integrations. Receivers can be email, PagerDuty, Slack etc. The Prometheus server reads the alert rules and if a condition for an alert is met, an alert is fired (i.e. sent to the alert manager).

### Prometheus Data Storage
Prometheus includes a local on-disk time series database, but optionally integrates with remote storage systems. Data in local storage is stored in a custom, highly efficient format.

### Querying Prometheus
Prometheus provides a functional query language called PromQL. It lets user select and aggregate time series data in real time.

Example queries:
```sh
http_requests_total{status!~"4.."}  # query all HTTP status codes except 4xx ones
rate(http_requests_total[5m])[30m:] # returns the 5min rate of the http_requests_total metric for the past 30mins
```

Options to view results are:
- Query target directly
- Prometheus Web UI
- Use a more powerful visualization tool, e.g. Grafana


</details>

*****

<details>
<summary>Video: 2 - Install Prometheus Stack in Kubernetes</summary>
<br />

### Setup Prometheus in an EKS Cluster
How can we deploy the different parts of the Prometheus monitoring stack in a Kubernetes cluster? There are several ways of doing it.

- Do it manually yoursef: Write all the configuration yaml files of the Prometheus components (like deployments, stateful sets, services, config maps, secrets etc. for Prometheus server, Alertmanager, Grafana, etc.) yourself and apply them to the cluster in the right order. This is a very inefficient way.
- Use an Operator: Find a Prometheus operator (manages the combination of all Prometheus components as one unit) and deploy it in the K8s cluster using the configuration files of the operator.
- Use a Helm chart to deploy an operator: Helm manages the initial setup of the operator and the operator will manage the running Prometheus setup.

In the [first demo project](./demo-projects/1-install-prometheus-in-k8s/) we 
- create an EKS cluster using eksctl,
- deploy the microservices application we know from module 10 (Kubernetes) on that cluster,
- deploy the Prometheus stack,
- monitor the Kubernetes cluster,
- and monitor the microservices application.

</details>

*****