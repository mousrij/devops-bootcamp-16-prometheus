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

#### Steps to deploy Redis exporter using Helm Chart
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

#### Steps to configure alert rules for Redis


#### Steps to import Grafana dashboard for Redis

