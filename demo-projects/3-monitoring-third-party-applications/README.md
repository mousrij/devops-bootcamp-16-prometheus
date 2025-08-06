## Demo Project - Monitoring Third-Party Applications

### Topics of the Demo Project
Configure Monitoring for a Third-Party Application

### Technologies Used
- Prometheus
- Kubernetes
- Redis
- Helm
- Grafana

### Project Description
Monitor Redis by using Prometheus Exporter
- Deploy Redis service in our cluster
- Deploy Redis exporter using Helm Chart
- Configure Alert Rules (when Redis is down or has too many connections)
- Import Grafana Dashboard for Redis to visualize monitoring data in Grafana


#### Steps to deploy Redis service in our cluster
The Redis service was already deployed as part of the online shop microservices application, so there's nothing left to do.
<details>
  <summary>####1- Steps to deploy Redis exporter using Helm Chart</summary>

The most popular Redis exporter is available at [https://github.com/oliver006/redis_exporter](https://github.com/oliver006/redis_exporter). Since we want to deploy it in a Kubernetes cluster we are going to use a Helm chart to do this.

There are different ways to find the chart:
- [ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus-redis-exporter)
- [Github](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter)

This chart bootstraps the deployment of exactly the Redis exporter mentioned above. In the [values.yaml](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-redis-exporter/values.yaml) file of the chart we can see what variables may be set/overridden when using this chart.

We need to know the name and port of the redis service in the K8s cluster:
```sh
kubectl get services
# NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice                                  ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       10d
# ...
# redis-exporter-prometheus-redis-exporter   ClusterIP      10.100.137.137   <none>                                                                      9121/TCP       43m
# rediscart                                  ClusterIP      10.100.206.38    <none>                                                                      6379/TCP       10d
# shippingservice                            ClusterIP      10.100.95.95     <none>                                                                      50051/TCP      10d
```

The service name is 'rediscart' and it is listening on port 6379. Now we can create a file called `redis-values.yaml` with the following content:
```yaml
serviceMonitor:
  enabled: true
  labels:
    release: monitoring

redisAddress: redis://rediscart:6379
```

```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# "prometheus-community" already exists with the same configuration, skipping

helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "ingress" chart repository
# ...Successfully got an update from the "bitnami" chart repository
# ...Successfully got an update from the "prometheus-community" chart repository
# Update Complete. ⎈Happy Helming!⎈

helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml
# NAME: redis-exporter
# LAST DEPLOYED: Sat Sep 16 22:31:50 2023
# NAMESPACE: default
# STATUS: deployed
# REVISION: 1
# TEST SUITE: None
# NOTES:
# 1. Get the Redis Exporter URL by running these commands:
#   export POD_NAME=$(kubectl get pods --namespace default -l "app=prometheus-redis-exporter,release=redis-exporter" -o jsonpath="{.items[0].metadata.name}")
#   echo "Visit http://127.0.0.1:8080 to use your application"
#   kubectl port-forward $POD_NAME 8080:

helm ls
# NAME          	NAMESPACE	REVISION	UPDATED                              	STATUS  	CHART                          	APP VERSION
# redis-exporter	default  	1       	2023-09-16 22:31:50.514561 +0200 CEST	deployed	prometheus-redis-exporter-5.6.0	v1.54.0 

kubectl get pods
# NAME                                                       READY   STATUS    RESTARTS   AGE
# adservice-599678658f-2r7w2                                 1/1     Running   0          10d
# ...
# redis-exporter-prometheus-redis-exporter-6595c4bf5-phxgq   1/1     Running   0          2m26s
# rediscart-f5cdf4c67-q9sdk                                  1/1     Running   0          10d
# ...

kubectl get servicemonitors
# NAME                                       AGE
# redis-exporter-prometheus-redis-exporter   5m20s

kubectl get servicemonitor redis-exporter-prometheus-redis-exporter -o yaml
# apiVersion: monitoring.coreos.com/v1
# kind: ServiceMonitor
# metadata:
#   annotations:
#     meta.helm.sh/release-name: redis-exporter
#     meta.helm.sh/release-namespace: default
#   creationTimestamp: "2023-09-16T20:31:51Z"
#   generation: 1
#   labels:
#     app.kubernetes.io/managed-by: Helm
#     release: monitoring                              <---------------
#   name: redis-exporter-prometheus-redis-exporter
#   namespace: default
#   resourceVersion: "1985765"
#   uid: bbff5ba5-263f-4823-b067-59010f374a75
# spec:
#   endpoints:
#   - port: redis-exporter
#   jobLabel: redis-exporter-prometheus-redis-exporter
#   namespaceSelector:
#     matchNames:
#     - default
#   selector:
#     matchLabels:
#       app.kubernetes.io/instance: redis-exporter
#       app.kubernetes.io/name: prometheus-redis-exporter
```

Open the Prometheus UI in the browser and select Status > Targets. You should see a new target called 'serviceMonitor/default/redis-exporter-prometheus-redis-exporter/0 (1/1 up)' with a 'Last Scrape' time of something like '28.127s ago' which means that Prometheus detected the new metrics endpoint and successfully scraped it. If you type 'redis' into the query execution input field, you should see all the metrics starting with redis (make sure the 'Enable autocomplete' checkbox is checked), e.g. 'redis_connected_clients'.
</details>

<details>
  <summary>#### Steps to configure alert rules for Redis</summary>
We want to be alerted when Redis is down or when it is running out of connections. We could write alert rules for these purposes on our own, or we can see if they are available in a public collection of already written useful alert rules:
- [https://github.com/samber/awesome-prometheus-alerts](https://github.com/samber/awesome-prometheus-alerts)
- [https://samber.github.io/awesome-prometheus-alerts/](https://samber.github.io/awesome-prometheus-alerts/)

The rules in the [redis section](https://samber.github.io/awesome-prometheus-alerts/rules#redis) are based on the metrics provided by the redis exporter we used and can therefore be used in our deployment. The first rule `RedisDown` is what we need, so lets copy it, create a file called `redis-rules.yaml` and paste it in there:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: redis-rules
  labels:
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
    - name: redis.rules
      rules:
      - alert: RedisDown
        expr: redis_up == 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: Redis down (instance {{ $labels.instance }})
          description: "Redis instance is down\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

The second rule we want to configure is the one called `RedisTooManyConnections`. It triggers an alert when more than 90% of the max allowed connection (configured in the redis configuration) are used. So lets copy that one too and add it to the `redis-rules.yaml` file:

```yaml
      - alert: RedisTooManyConnections
        expr: redis_connected_clients / redis_config_maxclients * 100 > 90
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: Redis too many connections (instance {{ $labels.instance }})
          description: "Redis is running out of connections (> 90% used)\n  VALUE = {{ $value }}\n  LABELS = {{ $labels }}"
```

Let's apply the rules to the K8s cluster:
```sh
kubectl apply -f redis-rules.yaml
# prometheusrule.monitoring.coreos.com/redis-rules created

kubectl get prometheusrules
# NAME          AGE
# redis-rules   4m24s
```

After some time you should see the new rules in the Prometheus UI (Alerts, redis.rules).

Now let's trigger the "RedisDown" alert by stopping the redis pod:
```sh
kubectl scale deployment rediscart --replicas=0
# deployment.apps/rediscart scaled
```

In the [values.yaml](https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-redis-exporter/values.yaml) file of the redis-exporter helm chart we can see in the 'serviceMonitor' section that the default scrape interval is '30s'. So after 30 seconds we should see in the Prometheus UI (Alerts) that the RedisDown alert is firing.

After restarting the redis pod with `kubectl scale deployment rediscart --replicas=1` and another 30 seconds, the alert state should go back to inactive.
</details>

<details>

  <summary>#### Steps to import Grafana dashboard for Redis</summary>
To better analyze an issue we were alerted about, we want to visualize the metrics data in Grafana. As with the alert rules we can either manually create a Redis dashboard in Grafana ourselves, or we can use a public available Grafana dashboard for Redis metrics.

On [Grafana Labs](https://grafana.com/grafana/dashboards/) you can look up existing dashboards. Search for 'redis' and make sure to select a dashboard that is based on the metrics of our [redis-exporter](https://github.com/oliver006/redis_exporter). In our case it is the ['Redis Dashboard for Prometheus Redis Exporter 1.x'](https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/). Copy the dashboard ID (763). Open the Grafana UI (http://localhost:8080/dashboards), click on the "+" button in the top right corner and select "Import dashboard". Paste the dashboard ID into the "Import via grafana.com" field and press "Load". Select "Prometheus" as the data source and press "Import".

The new dashboard gets displayed. It is connected to the correct instance (192.168.1.149:9121), which you can verify via the following commands:

```sh
kubectl get services
# NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice                                  ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       10d
# ...
# redis-exporter-prometheus-redis-exporter   ClusterIP      10.100.233.156   <none>                                                                      9121/TCP       13h
# rediscart                                  ClusterIP      10.100.206.38    <none>                                                                      6379/TCP       10d
# shippingservice                            ClusterIP      10.100.95.95     <none>                                                                      50051/TCP      10d

kubectl describe service redis-exporter-prometheus-redis-exporter
# Name:              redis-exporter-prometheus-redis-exporter
# Namespace:         default
# Labels:            app.kubernetes.io/instance=redis-exporter
#                    app.kubernetes.io/managed-by=Helm
#                    app.kubernetes.io/name=prometheus-redis-exporter
#                    app.kubernetes.io/version=v1.54.0
#                    helm.sh/chart=prometheus-redis-exporter-5.6.0
# Annotations:       meta.helm.sh/release-name: redis-exporter
#                    meta.helm.sh/release-namespace: default
# Selector:          app.kubernetes.io/instance=redis-exporter,app.kubernetes.io/name=prometheus-redis-exporter
# Type:              ClusterIP
# IP Family Policy:  SingleStack
# IP Families:       IPv4
# IP:                10.100.233.156
# IPs:               10.100.233.156
# Port:              redis-exporter  9121/TCP
# TargetPort:        exporter-port/TCP
# Endpoints:         192.168.1.149:9121           <---------------
# Session Affinity:  None
# Events:            <none>
```
</details>