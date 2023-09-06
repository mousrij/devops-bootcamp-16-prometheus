## Demo Project - Install Prometheus Stack in Kubernetes

### Topics of the Demo Project
Install Prometheus Stack in Kubernetes

### Technologies Used
- Prometheus
- Kubernetes
- Helm
- AWS EKS
- eksctl
- Grafana
- Linux

### Project Description
- Setup EKS cluster using eksctl
- Deploy a micreoservices application
- Deploy Prometheus, Alert Manager and Grafana in cluster as part of the Prometheus Operator using Helm chart


#### Steps to setup EKS cluster using eksctl
Execute the following commands (make sure the AWS configuration and credentials in `~/.aws` are set correctly):

```sh
# create a cluster in the default region with two worker nodes
eksctl create cluster

#2023-09-06 22:03:02 [ℹ]  eksctl version 0.141.0
#2023-09-06 22:03:02 [ℹ]  using region eu-central-1
#...
#2023-09-06 22:03:02 [ℹ]  building cluster stack "eksctl-ferocious-painting-1694030582-cluster"
#...
#2023-09-06 22:14:06 [ℹ]  building managed nodegroup stack "eksctl-ferocious-painting-1694030582-nodegroup-ng-1c743942"
#...
#2023-09-06 22:16:35 [✔]  saved kubeconfig as "/Users/fsiegrist/.kube/config"
#2023-09-06 22:16:35 [ℹ]  no tasks
#2023-09-06 22:16:35 [✔]  all EKS cluster resources for "ferocious-painting-1694030582" have been created
#2023-09-06 22:16:35 [ℹ]  nodegroup "ng-1c743942" has 2 node(s)
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-25-138.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-86-219.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-1c743942"
#2023-09-06 22:16:35 [ℹ]  nodegroup "ng-1c743942" has 2 node(s)
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-25-138.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:35 [ℹ]  node "ip-192-168-86-219.eu-central-1.compute.internal" is ready
#2023-09-06 22:16:36 [ℹ]  kubectl command should work with "/Users/fsiegrist/.kube/config", try 'kubectl get nodes'
#2023-09-06 22:16:36 [✔]  EKS cluster "ferocious-painting-1694030582" in "eu-central-1" region is ready


# when the cluster has been created (after about 10-15 minutes)
kubectl get nodes

# NAME                                              STATUS   ROLES    AGE     VERSION
# ip-192-168-25-138.eu-central-1.compute.internal   Ready    <none>   5m45s   v1.25.12-eks-8ccc7ba
# ip-192-168-86-219.eu-central-1.compute.internal   Ready    <none>   5m43s   v1.25.12-eks-8ccc7ba
```

#### Steps to deploy a Microservices Application
Apply the k8s config file `config-micorservices.yaml` to the cluster:

```sh
kubectl apply -f config-microservices.yaml
# deployment.apps/emailservice created
# service/emailservice created
# deployment.apps/recommendationservice created
# service/recommendationservice created
# deployment.apps/paymentservice created
# service/paymentservice created
# deployment.apps/productcatalogservice created
# service/productcatalogservice created
# deployment.apps/currencyservice created
# service/currencyservice created
# deployment.apps/shippingservice created
# service/shippingservice created
# deployment.apps/adservice created
# service/adservice created
# deployment.apps/cartservice created
# service/cartservice created
# deployment.apps/checkoutservice created
# service/checkoutservice created
# deployment.apps/frontend created
# service/frontend created
# deployment.apps/rediscart created
# service/rediscart created

kubectl get pods
# NAME                                     READY   STATUS    RESTARTS   AGE
# adservice-599678658f-2r7w2               1/1     Running   0          62s
# adservice-599678658f-p5vqq               1/1     Running   0          62s
# cartservice-7f96d4d6c9-fmhwp             1/1     Running   0          62s
# cartservice-7f96d4d6c9-m46ck             1/1     Running   0          62s
# checkoutservice-645cc49cd7-nq8jt         1/1     Running   0          62s
# checkoutservice-645cc49cd7-zftnc         1/1     Running   0          61s
# currencyservice-65d78749bb-4k7qg         1/1     Running   0          62s
# currencyservice-65d78749bb-bc9r5         1/1     Running   0          62s
# emailservice-556cd6f9b9-6cqhw            1/1     Running   0          63s
# emailservice-556cd6f9b9-jkw9b            1/1     Running   0          63s
# frontend-7dcff99cc6-68fsk                1/1     Running   0          61s
# frontend-7dcff99cc6-bw8tm                1/1     Running   0          61s
# paymentservice-695d749644-cwfs8          1/1     Running   0          63s
# paymentservice-695d749644-xl7qq          1/1     Running   0          63s
# productcatalogservice-8596c6c8cf-d896g   1/1     Running   0          62s
# productcatalogservice-8596c6c8cf-nzvtc   1/1     Running   0          62s
# recommendationservice-84f7fb567c-4c79j   1/1     Running   0          63s
# recommendationservice-84f7fb567c-g8dwx   1/1     Running   0          63s
# rediscart-f5cdf4c67-q9sdk                1/1     Running   0          61s
# shippingservice-59c76d497c-687kv         1/1     Running   0          62s
# shippingservice-59c76d497c-n9cr4         1/1     Running   0          62s

kubectl get services
# NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
# adservice               ClusterIP      10.100.22.140    <none>                                                                      9555/TCP       104m
# cartservice             ClusterIP      10.100.206.3     <none>                                                                      7070/TCP       104m
# checkoutservice         ClusterIP      10.100.215.205   <none>                                                                      5050/TCP       104m
# currencyservice         ClusterIP      10.100.96.68     <none>                                                                      7000/TCP       104m
# emailservice            ClusterIP      10.100.175.35    <none>                                                                      5000/TCP       104m
# frontend                LoadBalancer   10.100.57.134    aef508a483cb942da8affab91f564595-502614928.eu-central-1.elb.amazonaws.com   80:31545/TCP   104m
# kubernetes              ClusterIP      10.100.0.1       <none>                                                                      443/TCP        117m
# paymentservice          ClusterIP      10.100.208.157   <none>                                                                      50051/TCP      104m
# productcatalogservice   ClusterIP      10.100.141.240   <none>                                                                      3550/TCP       104m
# recommendationservice   ClusterIP      10.100.57.197    <none>                                                                      8080/TCP       104m
# rediscart               ClusterIP      10.100.206.38    <none>                                                                      6379/TCP       104m
# shippingservice         ClusterIP      10.100.95.95     <none>                                                                      50051/TCP      104m
```

Open the browser and navigate to 'http://aef508a483cb942da8affab91f564595-502614928.eu-central-1.elb.amazonaws.com' to see the running application.

#### Steps to deploy the Prometheus Stack using Helm
Execute the following commands:

```sh
# add the helm repository for prometheus 
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# "prometheus-community" already exists with the same configuration, skipping

helm repo update
# Hang tight while we grab the latest from your chart repositories...
# ...Successfully got an update from the "ingress" chart repository
# ...Successfully got an update from the "bitnami" chart repository
# ...Successfully got an update from the "prometheus-community" chart repository
# Update Complete. ⎈Happy Helming!⎈

# create a namespace for the monitoring (separate from the namespace the microservices application is deploed to)
kubectl create namespace monitoring

# install the chart
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
# NAME: monitoring
# LAST DEPLOYED: Wed Sep  6 22:31:19 2023
# NAMESPACE: monitoring
# STATUS: deployed
# REVISION: 1
# NOTES:
# kube-prometheus-stack has been installed. Check its status by running:
#   kubectl --namespace monitoring get pods -l "release=monitoring"
# 
# Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.

# check the status of the installed prometheus stack
kubectl --namespace monitoring get pods -l "release=monitoring"
# NAME                                                   READY   STATUS    RESTARTS   AGE
# monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj   1/1     Running   0          81s
# monitoring-kube-state-metrics-55b56ffff7-6m5mb         1/1     Running   0          81s
# monitoring-prometheus-node-exporter-7hkvl              1/1     Running   0          81s
# monitoring-prometheus-node-exporter-r8x8q              1/1     Running   0          81s

# or
kubectl get all -n monitoring
# NAME                                                         READY   STATUS    RESTARTS   AGE
# pod/alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          2m35s
# pod/monitoring-grafana-57b47fdd87-j762g                      3/3     Running   0          2m41s
# pod/monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj     1/1     Running   0          2m41s
# pod/monitoring-kube-state-metrics-55b56ffff7-6m5mb           1/1     Running   0          2m41s
# pod/monitoring-prometheus-node-exporter-7hkvl                1/1     Running   0          2m41s
# pod/monitoring-prometheus-node-exporter-r8x8q                1/1     Running   0          2m41s
# pod/prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          2m35s
# 
# NAME                                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
# service/alertmanager-operated                     ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   2m35s
# service/monitoring-grafana                        ClusterIP   10.100.22.92     <none>        80/TCP                       2m41s
# service/monitoring-kube-prometheus-alertmanager   ClusterIP   10.100.214.211   <none>        9093/TCP,8080/TCP            2m41s
# service/monitoring-kube-prometheus-operator       ClusterIP   10.100.79.107    <none>        443/TCP                      2m41s
# service/monitoring-kube-prometheus-prometheus     ClusterIP   10.100.145.159   <none>        9090/TCP,8080/TCP            2m41s
# service/monitoring-kube-state-metrics             ClusterIP   10.100.117.112   <none>        8080/TCP                     2m41s
# service/monitoring-prometheus-node-exporter       ClusterIP   10.100.141.223   <none>        9100/TCP                     2m41s
# service/prometheus-operated                       ClusterIP   None             <none>        9090/TCP                     2m35s
# 
# NAME                                                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
# daemonset.apps/monitoring-prometheus-node-exporter   2         2         2       2            2           kubernetes.io/os=linux   2m41s
# 
# NAME                                                  READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/monitoring-grafana                    1/1     1            1           2m41s
# deployment.apps/monitoring-kube-prometheus-operator   1/1     1            1           2m41s
# deployment.apps/monitoring-kube-state-metrics         1/1     1            1           2m41s
# 
# NAME                                                             DESIRED   CURRENT   READY   AGE
# replicaset.apps/monitoring-grafana-57b47fdd87                    1         1         1       2m42s
# replicaset.apps/monitoring-kube-prometheus-operator-7cc5877b9d   1         1         1       2m42s
# replicaset.apps/monitoring-kube-state-metrics-55b56ffff7         1         1         1       2m42s
# 
# NAME                                                                    READY   AGE
# statefulset.apps/alertmanager-monitoring-kube-prometheus-alertmanager   1/1     2m36s
# statefulset.apps/prometheus-monitoring-kube-prometheus-prometheus       1/1     2m36s
```

#### Understanding Prometheus Stack Components
There are

- two stateful sets
  - the Prometheus Server (`prometheus-monitoring-kube-prometheus-prometheus` created by the operator)
  - and the Alert Manager (`alertmanager-monitoring-kube-prometheus-alertmanager` created by the operator too)
- three deployments
  - the Prometheus operator (`monitoring-kube-prometheus-operator`) which created the two stateful sets mentioned above
  - Grafana (`monitoring-grafana`)
  - Kube State Metrics (`monitoring-kube-state-metrics`), which was created by its own Helm chart (which is a dependency of this Helm chart). It scrapes Kubernetes components, i.e. it monitors the health of deployments, stateful sets etc. inside the cluster and makes this information available for Prometheus. So you get the monitoring of the Kubernetes cluster out of the box.
- three replica sets related to the three deployments
- a node exporter daemon set (`monitoring-prometheus-node-exporter`), which runs on every worker node (= characteristic of daemon sets) and translates the worker node metrics to Prometheus metrics.
- several pods
- several services
- a bunch of config maps (`kubectl get configmaps -n monitoring`)
- some secrets (`kubectl get secrets -n monitoring`)
- and some CRDs (custom resource definition i.e. extension of the K8s API; `kubectl get crds -n monitoring`)

With the Prometheus monitoring stack the worker nodes and the Kubernetes components are monitored out of the box.

Components inside Prometheus, Alertmanager, Operator
To see what's inside Prometheus, Alertmanager and Operator, execute 
```sh
kubectl describe statefulset prometheus-monitoring-kube-prometheus-prometheus -n monitoring > prometheus-stack/prometheus.yaml
kubectl describe statefulset prometheus-monitoring-kube-prometheus-prometheus -n monitoring > prometheus-stack/alertmanager.yaml
kubectl describe deployment monitoring-kube-prometheus-operator -n monitoring > prometheus-stack/operator.yaml
```

In `prometheus-stack/prometheus.yaml` we can have a look at the containers. There is the `prometheus` container itself and its mounted volumes containing configuration files (what endpoints to scrape, alerting rules). The `config-reloader` is a side-car container in the pod and it's responsible to reload the configuration files when they change. So when we add a new target to be scraped or a new alert rule to consider, Prometheus will automatically reload the configuration without us having to restart Prometheus.

To get the content of the configurations, you have to find out which volume contains the configuration and which configmap or secret was the source for the volume.
_prometheus-stack/prometheus.yaml_
```yaml
...
  Containers:
   prometheus:
    ...
   config-reloader:
    ...
    Args:
      ...
      --config-file=/etc/prometheus/config/prometheus.yaml.gz # <--
      --watched-dir=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0 # <--
      ...
    Mounts:
      /etc/prometheus/config from config (rw)  # <--
      /etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0 from prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0 (rw)  # <--
      ...
  Volumes:
   config:  # <--
    Type:        Secret (a volume populated by a Secret)
    SecretName:  prometheus-monitoring-kube-prometheus-prometheus
    Optional:    false
    ...
   prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0:  # <--
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
    Optional:  false
...
```

```sh
kubectl get secret prometheus-monitoring-kube-prometheus-prometheus -n monitoring -o yaml > prometheus-stack/prometheus-config-secret.yaml
kubectl get configmap prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0 -n monitoring -o yaml > prometheus-stack/prometheus-rulefile-configmap.yaml
```

_prometheus-stack/prometheus-config-secret.yaml_
```yaml
apiVersion: v1
data: # <--
  prometheus.yaml.gz: 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
kind: Secret
metadata:
  creationTimestamp: "2023-09-06T20:31:41Z"
  labels:
    managed-by: prometheus-operator
  name: prometheus-monitoring-kube-prometheus-prometheus
  namespace: monitoring
  ownerReferences:
  - apiVersion: monitoring.coreos.com/v1
    blockOwnerDeletion: true
    controller: true
    kind: Prometheus
    name: monitoring-kube-prometheus-prometheus
    uid: 482d2485-329b-4941-997f-608ca0e05fbe
  resourceVersion: "4482"
  uid: 4e50aa8b-ed4e-436d-bdcb-b1e5d991d25f
type: Opaque
```

```sh
echo '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' | base64 -d | gunzip

# global:
#   evaluation_interval: 30s
#   scrape_interval: 30s
#   external_labels:
#     prometheus: monitoring/monitoring-kube-prometheus-prometheus
#     prometheus_replica: $(POD_NAME)
# rule_files:
# - /etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0/*.yaml
# scrape_configs:
# - job_name: serviceMonitor/monitoring/monitoring-kube-prometheus-alertmanager/0
#   honor_labels: false
#   kubernetes_sd_configs:
#   - role: endpoints
#     namespaces:
#       names:
#       - monitoring
#   metrics_path: /metrics
#   enable_http2: true
# ...
```

_prometheus-stack/prometheus-rulefile-configmap.yaml_
```yaml
apiVersion: v1
data:
  monitoring-monitoring-kube-prometheus-alertmanager.rules-13041404-8d85-473a-ac7f-2cab5f905f7b.yaml: |
    groups:
    - name: alertmanager.rules
      rules:
      - alert: AlertmanagerFailedReload
        annotations:
          description: Configuration has failed to load for {{ $labels.namespace }}/{{
            $labels.pod}}.
          runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagerfailedreload
          summary: Reloading an Alertmanager configuration has failed.
        expr: |-
          # Without max_over_time, failed scrapes could create false negatives, see
          # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
          max_over_time(alertmanager_config_last_reload_successful{job="monitoring-kube-prometheus-alertmanager",namespace="monitoring"}[5m]) == 0
        for: 10m
        labels:
          severity: critical
      - alert: AlertmanagerMembersInconsistent
        annotations:
          description: Alertmanager {{ $labels.namespace }}/{{ $labels.pod}} has only
            found {{ $value }} members of the {{$labels.job}} cluster.
          runbook_url: https://runbooks.prometheus-operator.dev/runbooks/alertmanager/alertmanagermembersinconsistent
          summary: A member of an Alertmanager cluster has not found all other cluster
            members.
        expr: |-
          # Without max_over_time, failed scrapes could create false negatives, see
          # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
            max_over_time(alertmanager_cluster_members{job="monitoring-kube-prometheus-alertmanager",namespace="monitoring"}[5m])
          < on (namespace,service,cluster) group_left
            count by (namespace,service,cluster) (max_over_time(alertmanager_cluster_members{job="monitoring-kube-prometheus-alertmanager",namespace="monitoring"}[5m]))
        for: 15m
        labels:
          severity: critical
    ...
```
