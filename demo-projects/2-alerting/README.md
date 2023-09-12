## Demo Project - Configure Alerting

### Topics of the Demo Project
Configure Alerting for our Application

### Technologies Used
- Prometheus
- Kubernetes
- Linux

### Project Description
Configure our Monitoring Stack to notify us whenever CPU usage exceeds 50% or a Pod cannot start.
- Configure Alert Rules in Prometheus Server
- Configure Alertmanager with Email Receiver


#### Steps to configure Alert Rules in Prometheus Server
Create a file called `alert-rules.yaml` and add the first alert rule definition (CPU usage exceeds 50%):
```yaml
name: HostHighCpuLoad
expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
for: 2m
labels:
  severity: warning
  namespace: monitoring
annotations:
  description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $label.instance }}\n"
  summary: Host CPU load high.
```

To develop the final value of the expression (`expr`), you can open the Prometheus UI, go to the main / home page ('http://localhost:9090/graph') and execute various expressions. For the example above you could test the following versions:
- `node_cpu_seconds_total{mode="idle"}`
- `rate(node_cpu_seconds_total{mode="idle"}[2m])`
- `(rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100`
- `avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100`
- `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)`

Now we want to add the new alert rule to the set of already configured rules. To find out, where the rules are stored, open the Prometheus UI and select Status > Configuration. The `rule_files` section looks like this:
```yaml
rule_files:
- /etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0/*.yaml
```

Fortunately we don't have to find the ConfigMap holding these rule files and adjust it (add our new rule to it). Since we have deployed Prometheus using the Prometheus Operator, there is a much simpler way. The Operator lets us define custom Kubernetes components (specified by CRDs), which contain alert rules. The operator will then tell Prometheus to reload its configuration.

To define a custom K8s component for alert rules, adjust the content of the `alert-rules.yaml` file as follows:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main-rules
  namespace: monitoring
  labels: # needed for the operator to find the components that need a configuration reload
    app: kube-prometheus-stack
    release: monitoring
spec:
  groups:
    - name: main.rules
      rules:
      - alert: HostHighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
        for: 2m
        labels:
          severity: warning
          namespace: monitoring
        annotations:
          description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $labels.instance }}\n"
          summary: Host CPU load high.
```

To find out the structure of the `spec` attribute, which is specific for each K8s resource, check the [documentation](https://docs.openshift.com/container-platform/4.13/rest_api/monitoring_apis/prometheusrule-monitoring-coreos-com-v1.html) of the custom component `PrometheusRule` (to find the documentation, do a Google search for 'PrometheusRule monitoring.coreos.com').

Now we add the second rule (Pod cannot start) to the file:
```yaml
      - alert: KubernetesPodCrashLooping
        expr: kube_pod_container_status_restarts_total > 5
        for: 0m
        labels:
          severity: critical
          namespace: monitoring
        annotations:
          description: "Pod {{ $labels.pod }} is crash looping\n Value = {{ $value }}"
          summary: Kubernetes pod crash looping.
```

Now we can apply the new component specification to the K8s cluster:
```sh
kubectl apply -f alert-rules.yaml
# prometheusrule.monitoring.coreos.com/main-rules created

kubectl get PrometheusRules -n monitoring
# NAME                                                              AGE
# main-rules                                                        54s  <--
# monitoring-kube-prometheus-alertmanager.rules                     6d
# monitoring-kube-prometheus-config-reloaders                       6d
# monitoring-kube-prometheus-etcd                                   6d
# monitoring-kube-prometheus-general.rules                          6d
# monitoring-kube-prometheus-k8s.rules                              6d
# ...
```

To find out whether the rules-configuration has been reloaded, we can inspect the logs of the according container:
```sh
kubectl get pods -n monitoring
# NAME                                                     READY   STATUS    RESTARTS   AGE
# alertmanager-monitoring-kube-prometheus-alertmanager-0   2/2     Running   0          6d
# monitoring-grafana-57b47fdd87-j762g                      3/3     Running   0          6d
# monitoring-kube-prometheus-operator-7cc5877b9d-t2zdj     1/1     Running   0          6d
# monitoring-kube-state-metrics-55b56ffff7-6m5mb           1/1     Running   0          6d
# monitoring-prometheus-node-exporter-7hkvl                1/1     Running   0          6d
# monitoring-prometheus-node-exporter-r8x8q                1/1     Running   0          6d
# prometheus-monitoring-kube-prometheus-prometheus-0       2/2     Running   0          6d

kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader
# level=info ts=2023-09-06T20:31:51.222379258Z caller=main.go:115 msg="Starting prometheus-config-reloader" version="(version=0.67.1, branch=refs/tags/v0.67.1, revision=ce782eafad7fb29bd4fa920052070007ae305013)"
# level=info ts=2023-09-06T20:31:51.22245098Z caller=main.go:116 build_context="(go=go1.20.6, platform=linux/amd64, user=Action-Run-ID-5750091714, date=20230803-11:24:04, tags=unknown)"
# level=info ts=2023-09-06T20:31:51.223938768Z caller=main.go:164 msg="Starting web server for metrics" listen=:8080
# level=info ts=2023-09-06T20:31:51.372475343Z caller=reloader.go:376 msg="Reload triggered" cfg_in=/etc/prometheus/config/prometheus.yaml.gz cfg_out=/etc/prometheus/config_out/prometheus.env.yaml watched_dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
# level=info ts=2023-09-06T20:31:51.372646384Z caller=reloader.go:237 msg="started watching config file and directories for changes" cfg=/etc/prometheus/config/prometheus.yaml.gz out=/etc/prometheus/config_out/prometheus.env.yaml dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
# level=info ts=2023-09-12T21:15:41.446366628Z caller=reloader.go:376 msg="Reload triggered" cfg_in=/etc/prometheus/config/prometheus.yaml.gz cfg_out=/etc/prometheus/config_out/prometheus.env.yaml watched_dirs=/etc/prometheus/rules/prometheus-monitoring-kube-prometheus-prometheus-rulefiles-0
```

We can see that a reload was triggered. So we should see our two new rules in the Prometheus UI under 'Alerts'.

To test the first alert rule ('HostHighCpuLoad') we are going to run a container simulating a CPU stress. Open the browser, navigate to [Docker Hub](https://hub.docker.com/) and search for 'cpustress'. Choose the [first serch result](https://hub.docker.com/r/containerstack/cpustress), copy the `docker run` command from the documentation and translate it to a `kubectl` command:
```sh
# docker
docker run -it --name cpustress --rm containerstack/cpustress --cpu 4 --timeout 150s --metrics-brief

# kubectl
kubectl run cpustress --image=containerstack/cpustress -- --cpu 4 --timeout 150s --metrics-brief
```

Run the `kubectl` command and open the Prometheus UI > Alerts. You can see that the state of the 'HostHighCpuLoad' rule is switching from 'inactive' to 'pending' and after 2 minutes it switches to 'firing'. After the cpustress pod has finished (150s) the CPU usage is decreasing and the state of the rule is going back to 'inactive'.

Delete the pod when you're done:
```sh
kubectl delete pod cpustress
```

#### Steps to configure Alertmanager with Email Receiver
