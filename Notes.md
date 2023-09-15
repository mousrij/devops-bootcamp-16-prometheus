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

<details>
<summary>Video: 3 - Data Visualization with Prometheus UI</summary>
<br />

With monitoring we want to notice when something unexpected happens, i.e. we want to observe anomalies (CPU spikes, insufficient storage, high load, unauthorized requests, etc.). And then we want to react accordingly. Visualizing the monitored data can help a lot in these tasks.

### Prometheus UI
After having deployed the Prometheus monitoring stack, you can setup port forwarding for the service `service/monitoring-kube-prometheus-prometheus`:

```sh
kubectl port-forward service/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
# [1] 20532
# Forwarding from [::1]:9090 -> 9090
``` 

Open the browser and navigate to 'http://localhost:9090' to see the Prometheus Web UI.
- To see the list of monitored components, open Status (in the menu bar) > Targets.
- To see all the metrics being collected, press the 'earth'-button next to "Execute" on the top right of the Prometheus dashboard and select a metric. To filter a metric you can add the filter criteria within curly braces (e.g. `apiserver_request_total{verb="GET"}`). 
- To see the configuration, open Status > Configuration. The `scrape_configs` section within the configuration contains a list of jobs:

```yaml
...
scrape_configs:
- job_name: serviceMonitor/monitoring/monitoring-kube-prometheus-alertmanager/0
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  follow_redirects: true
  enable_http2: true
  ...
```

What is the concept of a `job` in Prometheus? A target may have multiple endpoints (aka instances). And a collection of such instances with the same purpose is called a 'job'. 

Prometheus UI can be helpful to debug the configuration. But it is not really helpful in visualizing anomalies. Grafana, which is discussed in the next video, is much better for that purpose.

</details>

*****

<details>
<summary>Video: 4 - Introduction to Grafana</summary>
<br />

### Introduction to the Grafana UI
Grafana is a powerful open source visualization and analytics software. It was already deployed with the Prometheus Operator Helm Chart.

List the services in the 'monitoring' namespace to get information about the Grafana service:
```sh
kubectl get services -n monitoring
# NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
# alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2d
# monitoring-grafana                        ClusterIP   10.100.22.92     <none>        80/TCP                       2d
# monitoring-kube-prometheus-alertmanager   ClusterIP   10.100.214.211   <none>        9093/TCP,8080/TCP            2d
# monitoring-kube-prometheus-operator       ClusterIP   10.100.79.107    <none>        443/TCP                      2d
# monitoring-kube-prometheus-prometheus     ClusterIP   10.100.145.159   <none>        9090/TCP,8080/TCP            2d
# monitoring-kube-state-metrics             ClusterIP   10.100.117.112   <none>        8080/TCP                     2d
# monitoring-prometheus-node-exporter       ClusterIP   10.100.141.223   <none>        9100/TCP                     2d
# prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     2d
```

As you see the 'monitoring-grafana' service is listening on port 80. So lets setup port forwarding for this service:

```sh
kubectl port-forward service/monitoring-grafana -n monitoring 8080:80
# [2] 22356
# Forwarding from [::1]:8080 -> 3000
``` 

Open the browser and navigate to 'http://localhost:8080' to get to the login page of Grafana. The Helm chart of Prometheus stack is configured with login credentials. The username is 'admin' and the password is 'prom-operator'.

The Grafana UI is made up of dashboards. A dashboard is a set of one or more panels. Toggle the hamburger menu on the top left and select 'Dashboards' to open a list of dashboards already configured by the Helm chart. Dashboards are grouped into folders ('General' is the only folder right now). The get a first impression, open the 'Kubernetes / Compute Resources / Cluster' dashboard. 

As you see, a dashboard is a set of one or more panels. It is organized into one or more rows. Rows are logical dividers within a dashboard. They are used to group panels together.

Panel are the basic visualization building blocks in Grafana. They are composed by a query and a visualization. Each panel has a query editor specific to the data source selected in the panel. A panel can be moved and resized within a dashboard.

you can create your own dashboard if you want or you can edit existing ones (e.g. adding / removing panels).

Using the time range selector in the upper right you can zoom in or out in a graph. To zoom in you can also directly select the time frame of interest in a displayed graph.

In the right upper corner of each panel you'll find a 3-dot menu providing action like 'Edit', where you can edit the PromQL expressions used to collect the metrics the panel is based on. You may also choose another chart type.

### Create your own Dashboard
Navigate to Home > Dashboards, press the blue "New" button and select "New dashboard". Press "Add visualization" and select the Prometheus data source. In the metrics explorer you can select a metric, e.g. `cluster:node_cpu:sum_rate5m`. Press "Run queries" to display the graph. Press "Apply" to put the visualization on your new dashboard. Using the "Add" button you can add new rows or visualizations to the dashboard.

See [Grafana Documentation](https://grafana.com/docs/grafana/latest/dashboards/)

### Test Anomalies
To force a CPU spike that can be analyzed in Grafana, we are going to deploy a pod in the cluster, that sends a lot of requests to the endpoint of our online shop application. Execute the following command:
```sh
kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
[ root@curl-test:/ ]$ vi test.sh
# add the following content in vi:
for i in $(seq 1 10000)
do
  curl http://aef508a483cb942da8affab91f564595-502614928.eu-central-1.elb.amazonaws.com > text.txt
done
# save the file and make it executable
chmod +x test.sh

# run the script
./test.sh

# after the script has ended (ca. 5min) exit the pod
exit
```

Now you can have a look at the dashboards `Node Exporter / Nodes`, `Kubernetes / Compute Resources / Cluster`, `Kubernetes / Compute Resources / Namespace (Workloads)`, `Kubernetes / Compute Resources / Node (Pods)` or `Kubernetes / Compute Resources / Pod` to see the spike in CPU usage and find out which microservice contributed the biggest part of this CPU usage.

### Configure Users & Data Sources
As an admin user you can manage teams and users in Grafana. Open the hamburger menu in the top left corner and select 'Administration > Users' or 'Administration > Teams'.

To manage the datasources (endpoints Grafana uses to collect its data), open the hamburger menu in the top left corner and select 'Connections > Add new connection' or 'Connections > Data sources'. In the dashboards you can only visualize data coming from one of the configured data sources.

In the current Grafana setup configured by the Helm chart, there are two data sources: 'Prometheus' (= default) with the endpoint http://monitoring-kube-prometheus-prometheus.monitoring:9090 and 'Alertmanager' with the endpoint http://monitoring-kube-prometheus-alertmanager.monitoring:9093/.


</details>

*****

<details>
<summary>Video: 5 - Alert Rules in Prometheus</summary>
<br />

Instead of constantly checking the metrics visualization, you want to get notified when something special happens. Only then you will check your dashboards. So we need to configure the monitoring stack to notify us whenever something unexpected happens.

Alerting with Prometheus is separated into 2 parts:
- 1. Alerting rules in Prometheus server send alerts to an Alertmanager
- 2. Alertmanager then manages (deduplicating, grouping, routing) those alerts, including sending out notifications

Main steps to setup alerting and notifications:
- Setup and configure the Alertmanager
- Configure Prometheus to talk to the Alertmanager
- Create alerting rules in Prometheus

Prometheus server and Alertmanager each have their own configuration file.

See [Prometheus Alerting Documentation](https://prometheus.io/docs/alerting/latest/overview/)

### Examine existing alert rules
Existing alert rules can inspect on Prometheus UI > Alerts. The first one looks like this:
```yaml
name: AlertmanagerFailedReload
expr: max_over_time(alertmanager_config_last_reload_successful{job="monitoring-kube-prometheus-alertmanager",namespace="monitoring"}[5m]) == 0
for: 10m
labels:
  severity: critical
annotations:
  description: Configuration has failed to load for {{ $labels.namespace }}/{{ $labels.pod}}.
  runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload
  summary: Reloading an Alertmanager configuration has failed.
```

`expr` is a PromQL expression. It contains a metric (`alertmanager_config_last_reload_successful`), a filter applied to that metric (`{job="monitoring-kube-prometheus-alertmanager",namespace="monitoring"}`) and a function applied on the filtered metric (`max_over_time(...[5m])`, [5m] is a parameter of the function).

`severity` let's you define how important/severe the problem is.\
(By the way: You may define your own custom labels to group alert rules and select them based on this grouping.)

`for` lets you define a duration Prometheus is waiting before it fires an alert if the problem still exists after that duration.

`description` specifies the alert message that will be sent. `$labels` references the metric attributes (that can be used to filter the metric).

`runbook_url` points to a web page describing the alert rule (including tips on how to fix the problem if it occurs).

`summary` holds a short explanation of the alert rule.

</details>

*****

<details>
<summary>Video: 6, 7, 8 - Create own Alert Rules</summary>
<br />

Let's say we want to get an alert 
- when the CPU usage exceeds 50% (based on the Grafana visualization we saw that normally the CPU usage is between 20 and 40%)
- or when a Pod cannot start.

How do we configure according alert rules?

See the first part of [demo project #2](./demo-projects/2-alerting/).

</details>

*****

<details>
<summary>Video: 9 - Introduction to Alertmanager</summary>
<br />

When a Prometheus alerting rule is in 'firing' state, it means that Prometheus is sending the alert to the alertmanager. The alertmanager then dispatches notifications about the alert. It takes care of deduplicating, grouping and routing the alerts to the correct receiver integration.

Alertmanager is an application on its own and thus has its own configuration file. It also has a simple UI, which can be accessed after doing the following port forwarding:

```sh
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093
```

Select 'Status' to get some status information as well as the current configuration:

```yaml
global:
  resolve_timeout: 5m
  http_config:
    follow_redirects: true
    enable_http2: true
  smtp_hello: localhost
  smtp_require_tls: true
  pagerduty_url: https://events.pagerduty.com/v2/enqueue
  opsgenie_api_url: https://api.opsgenie.com/
  wechat_api_url: https://qyapi.weixin.qq.com/cgi-bin/
  victorops_api_url: https://alert.victorops.com/integrations/generic/20131114/alert/
  telegram_api_url: https://api.telegram.org
  webex_api_url: https://webexapis.com/v1/messages
route:
  receiver: "null"  # <-- global receiver definition
  group_by:
  - namespace
  continue: false
  routes:           # <-- receivers for specific alerts
  - receiver: "null"
    matchers:
    - alertname=~"InfoInhibitor|Watchdog"  # <-- label=value
    continue: false
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h  # <-- deduplication
inhibit_rules:
- source_matchers:
  - severity="critical"
  target_matchers:
  - severity=~"warning|info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - severity="warning"
  target_matchers:
  - severity="info"
  equal:
  - namespace
  - alertname
- source_matchers:
  - alertname="InfoInhibitor"
  target_matchers:
  - severity="info"
  equal:
  - namespace
receivers:
- name: "null"
templates:
- /etc/alertmanager/config/*.tmpl
```

The main sections of the alertmanager configuration file are:
- `global`: parameters that are valid in all other configuration contexts
- `route`: defines which alerts are dispatched to which receiver; only receivers declared in the `receivers` section may be referenced here
- `inhibit_rules`
- `receivers`: defines the receivers (notification integrations); only 
- `templates`

As you can see, the only receiver configured by default is the "null" receiver. That's why no notifications were sent to anywhere even though Prometheus was firing alerts.

For each alert you can define its own receiver. For example:
- send all K8s cluster related issues to admin email
- send all application related issues to developer team's slack channel
This is configured in the `route` section.

</details>

*****

<details>
<summary>Video:  10 - Configure Alertmanager with Email Receivers</summary>
<br />

We now want to configure an email receiver for the two alert rules.

See the second part of [demo project #2](./demo-projects/2-alerting/).

</details>

*****

<details>
<summary>Video:  11 - Trigger Alerts for Email Receiver</summary>
<br />

We now want to trigger alerts to test the email receiver.

See the finish of [demo project #2](./demo-projects/2-alerting/).

</details>

*****