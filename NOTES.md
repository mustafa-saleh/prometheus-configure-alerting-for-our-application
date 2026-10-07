# 16 - Monitoring with Prometheus

## 1 - Introduction to Monitoring with Prometheus | 20:55

Prometheus is an open-source monitoring and alerting toolkit designed for reliability and scalability. It collects metrics from configured targets at specified intervals, evaluates rule expressions, displays results, and can trigger alerts if certain conditions are met.

Prometheus is widely used in cloud-native environments, especially with Kubernetes, due to its powerful querying language (PromQL) and its ability to integrate with various visualization tools like Grafana.

Prometheus server consists of several components:

- **Retrieval component**, which scrapes metrics from configured targets.
- **Time series database**, which stores all the metrics data.
- **HTTP server**, which serves the Prometheus web UI and API.

Prometheus metrics have "HELP" and "TYPE" metadata, which provide information about the metric and its type (e.g., counter, gauge, histogram).

- **counter**: A metric that represents a single monotonically increasing counter, typically used for counting events.
- **gauge**: A metric that represents a single numerical value that can arbitrarily go up and down, such as temperature or current memory usage.
- **histogram**: A metric that samples observations (usually things like request durations or response sizes) and counts them in configurable buckets.

Prometheus pull data from targets (hostname/metrics endpoint) at regular intervals, and it can also support a push model through the use of the Pushgateway for ephemeral jobs. Data must be in a format that Prometheus understands, typically exposed via HTTP in a plaintext format.

Some applications have native prometheus support, while others may require the use of exporters to expose their metrics. Exporters are separate processes that collect metrics from a service and expose them in a format that Prometheus can scrape.

Newrelic & AWS Cloud Watch rely on the servers/applications to push metrics to them which may overload the network with requests, while Prometheus relies on a pull model where it scrapes metrics from the targets.

Prometheus offers:

- **Prometheus Server**: The core component that scrapes and stores metrics data.
- **Exporters**: These are used to expose metrics from various services and applications in a format that Prometheus can scrape.
- **Alertmanager**: Handles alerts sent by the Prometheus server and manages alert notifications.
- **Pushgateway**: Allows ephemeral and batch jobs to expose their metrics (push) to Prometheus.
- **Service Discovery**: Prometheus can automatically discover targets to scrape metrics from, especially in dynamic environments like Kubernetes.
- **PromQL**: Prometheus Query Language, used to query and aggregate metrics data.

Prometheus has a config file (prometheus.yml) that defines:

- **global settings**: such as scrape interval and evaluation interval.
- **scrape_configs**: which define the targets to scrape metrics from, including job names and static or dynamic target configurations.
- **alerting rules**: which define conditions for triggering alerts

Prometheus has it's own /metrics endpoint that exposes its own metrics, which can be scraped by another Prometheus server for monitoring Prometheus itself.

## 2 - Install Prometheus Stack in Kubernetes | 20:57

Prometheus can be installed in a Kubernetes cluster using Helm, which simplifies the deployment process. The Prometheus stack typically includes Prometheus server, Alertmanager, and Grafana for visualization.

Create an EKS cluster with default settings and install Prometheus stack using Helm:

```bash
# create an EKS cluster using eksctl with 2 nodes and default settings
eksctl create cluster

# deploy the online shop microservices application to the EKS cluster
kubectl apply -f config-microservices.yaml
```

Install prometheus stack using Helm:

```bash
# add prometheus-community helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
kubectl get all -n monitoring
kubectl get configmap -n monitoring
kubectl get secret -n monitoring
```

## 3 - Data Visualization with Prometheus UI | 11:11

Prometheus provides a web-based user interface that allows users to explore and visualize metrics data. The UI can be accessed by port-forwarding the Prometheus service to a local port.

```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring &
```

Open the browser and navigate to `http://localhost:9090` to access the Prometheus UI. From here, you can use the "Graph" tab to query metrics using PromQL and visualize the results.

Prometheus job is a collection of instances with the same purpose, and each instance is a target that Prometheus scrapes metrics from. The job name is used to group related targets together.

## 4 - Introduction to Grafana | 23:25

Grafana is an open-source platform for monitoring and observability that allows users to query, visualize, alert on, and understand their metrics no matter where they are stored. It integrates with various data sources, including Prometheus, and provides a rich set of visualization options.

```bash
# localhost:8080 user: admin password: prom-operator
kubectl port-forward svc/monitoring-grafana 8080:80 -n monitoring &
# or get admin password from secret
kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

Grafana allows users to create dashboards, panels, and alerts based on the metrics data collected by Prometheus. It supports various visualization types, including graphs, tables, heatmaps, and more.

To access Grafana, open the browser and navigate to `http://localhost:8080`. The default login credentials are `admin` for the username and `prom-operator` for the password. After logging in, you can configure data sources, create dashboards, and set up alerts based on your monitoring needs.

To generate a CPU spike in the cluster, let's deploy a pod that will send large number of requests to the online shop microservices application. 

```bash
# radial/busyboxplus is very old
kubectl run curl-test --image=radial/busyboxplus:curl -i --tty --rm
# or curl images
kubectl run curl-test --image=curlimages/curl -i --tty --rm -- sh
```

To the appplication endpoint, check the frontend external service URL using `kubectl get svc` and send requests to it using the following command:

Create "test.sh" file with the following content:

```bash
for i in $(seq 1 10000)
do
  curl a1e9d040ec01f43ca981221bb724f477-1321322907.us-east-1.elb.amazonaws.com > test.txt
done
```

Run the script in the pod:

```bash
chmod +x test.sh
./test.sh
```

Check the Grafana dashboard (Kubernetes / Compute Resources / Cluster) to see the CPU spike in the cluster. For pod breakdown check the dashboard (Kubernetes / Compute Resources / Node (Pods)) to check which pods are consuming the most CPU resources. 

We can configure user in Grafana under "Users and Access" and the data sources for the dashboards under "connections -> data sources".

## 5 - Alert Rules in Prometheus | 20:22

Prometheus allows users to define alerting rules based on specific conditions in the metrics data. These rules can trigger alerts when certain thresholds are met, such as high CPU usage or low memory availability.

Example alerting rule for high CPU usage:

```yaml
# Define a group of alert rules
groups:
  # Name of the alert group
  - name: example
    # List of rules within this group
    rules:
      # Alert rule name
      - alert: HighCPUUsage
        # PromQL expression: calculate sum of CPU rate over 5m window grouped by pod, alert if > 80%
        expr: sum(rate(container_cpu_usage_seconds_total{image!="",container!="POD"}[5m])) by (pod) > 0.8
        # Duration the condition must be true before alerting (2 minutes)
        for: 2m
        # Labels assigned to the alert for organization and routing
        labels:
          severity: warning
        # Additional metadata for the alert notification
        annotations:
          # Short summary of the alert
          summary: "High CPU usage detected for pod {{ $labels.pod }}"
          # Detailed description of what the alert means
          description: "Pod {{ $labels.pod }} is using more than 80% of CPU for the last 2 minutes."
```

## 6 - Create own Alert Rules - Part 1 | 9:31

```yaml
name: HostHighCpuLoad
expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 50
for: 2m
labels:
  severity: warning
  namespace: monitoring
annotations:
  description: "CPU load on host is over 50%\n Value = {{ $value }}\n Instance = {{ $labels.instance }}\n"
  summary: "Host CPU load high"
```

##  7 - Create own Alert Rules - Part 2 | 20:45

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: main.rules
  namespace: monitoring
  # labels for prometheus operator to identify this resource as a PrometheusRule & load it into Prometheus
  labels:
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
        summary: "Host CPU load high"
    - alert: KubernetesPodCrashLooping
      expr: kube_pod_container_status_restarts_total > 5
      for: 0m
      labels:
        severity: critical
        namespace: monitoring
      annotations:
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is crash looping.\n Value = {{ $value }}\n Instance = {{ $labels.instance }}\n"
        summary: "Kubernetes Pod Crash Looping"
```

Apply the rule with the following command:

```bash
kubectl apply -f alert-rules.yaml
kubectl get prometheusrules -n monitoring
```

Prometheus has a sidecar container to reload the config, to check the logs of the prometheus pod to see if the config has been reloaded:

```bash
kubectl get pods -n monitoring
# -c config-reloader to specify which container logs you're interested in, as the prometheus pod has multiple containers
kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c config-reloader
# to check prometheus logs 
kubectl logs prometheus-monitoring-kube-prometheus-prometheus-0 -n monitoring -c prometheus
```

Check the Prometheus UI under "Alerts" tab to see the new alert rules and their status. You can also test the alerting rules by generating load on the cluster and observing if the alerts are triggered as expected.

## 8 - Create own Alert Rules - Part 3 | 5:39

Simulate a CPU spike to trigger the HostHighCpuLoad alert rule. You can use a stress testing tool or a simple script to generate CPU load on the nodes in your Kubernetes cluster.

Check Grafana dashboard (Kubernetes / Compute Resources / Cluster) to see the CPU spike in the cluster. Edit the "CPU Utilization" panel to display the usage as percentage with the following query:

```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)
``` 

From the right side menu, under "Standard options" set the "Unit" to "percent (0-100)".

In docker hub, search for "cpustress" to find a docker image that can be used to generate CPU load. Use the following command to run the container and generate CPU load:

```bash
kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 30s --metrics-brief
```

Check the Prometheus UI under "Alerts" tab to see if the HostHighCpuLoad alert is triggered & check Grafana dashboard to see the CPU spike in the cluster. You can also check the Prometheus logs to see if the alert was triggered and sent to Alertmanager.

## 9 - Introduction to Alertmanager | 8:34

When an alert is fired in Prometheus, it is sent to the Alertmanager, which is responsible for managing and routing alerts to the appropriate receivers based on the alert's labels and configured routing rules.

Check the alert manager UI by port-forwarding the Alertmanager service to a local port:

```bash
# localhost:9093
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring &
```

The alert manager default config doesn't have any receivers configured, so we need to create a config map with the alert manager configuration and apply it to the cluster.

## 10 - Configure Alertmanager with Email Receiver | 22:06

The prometheus operator extends the k8s api and add custom components that can be used to configure prometheus and alertmanager. The prometheus operator has a custom resource called "Alertmanager" that can be used to configure the alertmanager.

Create "alert-manager-configuration.yaml" file with the following content:

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: main-rules-alert-config
  namespace: monitoring
spec:
  route:
    receiver: 'email'
    repeatInterval: 30m
    routes:
    - matchers:
      - name: alertname
        value: HostHighCpuLoad
    - matchers:
      - name: alertname
        value: KubernetesPodCrashLooping
      repeatInterval: 10m
  receivers:
  - name: 'email'
    emailConfigs:
    - to: 'nana.janashia@nnsoftware.at'
      from: 'nana.janashia@nnsoftware.at'
      smarthost: 'smtp.gmail.com:587'
      authUsername: 'nana.janashia@nnsoftware.at'
      authIdentity: 'nana.janashia@nnsoftware.at'
      authPassword:
        name: gmail-auth
        key: password
```

Create the "email-secret.yaml" file with the following content:

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: gmail-auth
  namespace: monitoring
data:
  password: base64-encoded-value-of-your-password
```

To create an app password for Gmail, go to your Google account settings, navigate to "Security", and under "Signing in to Google", select "App passwords" or visit the url https://myaccount.google.com/apppasswords. Generate a new app password for "Mail" and use it in the secret.

Apply the secret and the alert manager configuration to the cluster:

```bash
kubectl apply -f email-secret.yaml
kubectl apply -f alert-manager-configuration.yaml
kubectl get alertmanagerconfig -n monitoring
# check config reload
kubectl get pods -n monitoring
kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c config-reloader
```

Check the alert manager UI to see if the configuration is loaded correctly. You can also check the Prometheus UI under "Alerts" tab to see if the alerts are firing and being sent to Alertmanager.

Alert manager config docs can be found at https://docs.redhat.com/en/documentation/openshift_container_platform/4.16/html/monitoring_apis/alertmanagerconfig-monitoring-coreos-com-v1beta1

## 11 - Trigger Alerts for Email Receiver | 6:06

Restart the cpu-test pod to generate CPU load and trigger the HostHighCpuLoad alert. Check the Prometheus UI under "Alerts" tab to see if the alert is firing and being sent to Alertmanager. Check your email inbox for the alert notification.

```bash
kubectl delete pod cpu-test
kubectl run cpu-test --image=containerstack/cpustress -- --cpu 4 --timeout 60s --metrics-brief
```

The alert manager have an endpoint (localhost:9093/api/v2/alerts) that can be used to check the alerts that are currently firing. You can use curl or any other HTTP client to check the alerts.

```bash
curl -X GET http://localhost:9093/api/v2/alerts
```

If the email receiver is not receiving the alerts, check the alert manager logs to see if there are any errors in sending the email. You can also check the Prometheus logs to see if the alert is being sent to Alertmanager.

```bash
kubectl logs alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -c alertmanager
```

## 12 - Monitor Third-Party Applications | 2:46

To monitor third-party applications, you can use exporters that expose metrics in a format that Prometheus can scrape. For example, to monitor Redis, you can use the Redis exporter. We need to create a ServiceMonitor resource to tell Prometheus to scrape metrics from the Redis exporter. The ServiceMonitor resource is a custom resource provided by the Prometheus Operator that allows you to define how to monitor a service.

```bash
kubectl get servicemonitors -n monitoring
```

## 13 - Deploy Redis Exporter | 14:32

We can install the Redis exporter using Helm https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter. First, we need to add the Helm repository for the Redis exporter and then install it in the monitoring namespace.

Create "redis-values.yaml" file with the following content:

```yaml
serviceMonitor:
  enabled: true
  labels:
    release: monitoring

redisAddress: redis://redis-cart:6379
```

Install the chart using the following command:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install redis-exporter prometheus-community/prometheus-redis-exporter -f redis-values.yaml 
```

Check the deployed components using the following command:

```bash
kubectl get pod
kubectl get servicemonitor
```

In prometheus UI, check the "Targets" tab to see if the Redis exporter is being scraped and if the metrics are being collected. You can also check the "Graph" tab to query and visualize the Redis metrics.

If you search "redis" in the Prometheus UI, you should see metrics related to Redis exposed by redis exporter, such as `redis_up`, `redis_connected_clients`, `redis_memory_used_bytes`, etc. You can use these metrics to create dashboards in Grafana and set up alerts in Prometheus based on the Redis metrics.

## 14 - Alert Rules & Grafana Dashboard for Redis | 13:15

We can use the redis metrics exposed by the Redis exporter to create alert rules in Prometheus and dashboards in Grafana. For example, we can create an alert rule for when the Redis server is down or when the number of connected clients exceeds a certain threshold.

Awesome Prometheus Rules "https://samber.github.io/awesome-prometheus-alerts/" contains ready to use alert rules for various applications, including Redis. We can use the following alert rules for Redis:

Create "redis-rules.yaml" file with the following content:

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
    - alert: RedisTooManyConnections
      expr: redis_connected_clients / redis_config_maxclients * 100 > 90
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: Redis too many connections (instance {{ $labels.instance }})
        description: "Redis has {{ $value }} connections\n  LABELS = {{ $labels }}"
```

Apply the alert rules to the cluster:

```bash
kubectl apply -f redis-rules.yaml
kubectl get prometheusrules 
```

To trigger the "Redis Down" alert, we can delete the Redis pod and check the Prometheus UI under "Alerts" tab to see if the alert is firing. We can also check the Alertmanager UI to see if the alert is being sent to the email receiver.

```bash
# set replicas to 0 to simulate Redis down
kubectl edit deployment redis-cart
kubectl get pod
```

After 30 seconds (scrape interval according to docs https://github.com/prometheus-community/helm-charts/blob/main/charts/prometheus-redis-exporter/values.yaml), the "Redis Down" alert should be triggered and sent to the email receiver. Check your email inbox for the alert notification.

We can create a Grafana dashboard for Redis using the metrics exposed by the Redis exporter. We can create panels for metrics such as `redis_up`, `redis_connected_clients`, `redis_memory_used_bytes`, etc. and visualize them in a dashboard.

The Grafana Labs website "https://grafana.com/grafana/dashboards/763-redis-dashboard-for-prometheus-redis-exporter-1-x/" has a collection of pre-built dashboards for various applications, including Redis. We can import the Redis dashboard with ID 763 into Grafana to visualize the Redis metrics.

To import the dashboard, go to Grafana UI, click on the "+" icon on the left sidebar, select "Import", and enter the dashboard ID 763. Click "Load" and then select the Prometheus data source to import the dashboard.

## 15 - Collect & Expose Metrics with Prometheus Client Library (Monitor own App - Part 1) | 22:12

So far we have implemented monitoring for:

- Resource Consumption (CPU, Memory, Disk, Network) on Nodes
- Kubernetes Cluster Components (Pods, Deployments, Services)
- Prometheus Stack
- Third-Party Applications (Redis)
- Now we will implement monitoring for our own application (Online Shop Microservices) using Prometheus Client Library.

Prometheus Client Libraries are available for various programming languages, including Python, Java, Go, and Node.js. These libraries allow developers to instrument their applications and expose custom metrics that can be scraped by Prometheus.

Let's use the following NodeJs app as an example:

```javascript
const express = require('express');
const path = require('path');
const fs = require('fs');
const app = express();

const client = require('prom-client');
const collectDefaultMetrics = client.collectDefaultMetrics;
// Probe every 5th second.
collectDefaultMetrics({ timeout: 5000 });

const httpRequestsTotal = new client.Counter({
  name: 'http_request_operations_total',
  help: 'Total number of Http requests'
})

const httpRequestDurationSeconds = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of Http requests in seconds',
  buckets: [0.1, 0.5, 2, 5, 10]
})

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType)
  res.end(await client.register.metrics())
})

app.get('/', function (req, res) {
    // Simulate sleep for a random number of milliseconds
    var start = new Date()
    var simulateTime = Math.floor(Math.random() * (10000 - 500 + 1) + 500)

    setTimeout(function(argument) {
      // Simulate execution time
      var end = new Date() - start
      httpRequestDurationSeconds.observe(end / 1000); //convert to seconds
    }, simulateTime)

    httpRequestsTotal.inc();
    res.sendFile(path.join(__dirname, "index.html"));
});

app.get('/profile-picture-andrea', function (req, res) {
  const img = fs.readFileSync(path.join(__dirname, "images/profile-andrea.jpg"));
  res.writeHead(200, {'Content-Type': 'image/jpg' });
  res.end(img, 'binary');
});

app.get('/profile-picture-ari', function (req, res) {
  const img = fs.readFileSync(path.join(__dirname, "images/profile-ari.jpeg"));
  res.writeHead(200, {'Content-Type': 'image/jpg' });
  res.end(img, 'binary');
});

app.listen(3000, function () {
  console.log("app listening on port 3000!");
});
```

Create a docker image & push it to DockerHub:

```bash
docker build -t mustafa199b/demo:nodeapp .
docker login
docker push mustafa199b/demo:nodeapp
```

Deploy the app to the cluster, create secret to connect to DockerHub and create deployment and service for the app:

```bash
# create secret to connect to DockerHub
kubectl create secret docker-registry my-registry-key --docker-server=https://index.docker.io/v1/ --docker-username=mustafa199b --docker-password=<your-dockerhub-password> 
```

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  selector:
    matchLabels:
      app: nodeapp
  template:
    metadata:
      labels:
        app: nodeapp
    spec:
      imagePullSecrets:
      - name: my-registry-key
      containers:
      - name: nodeapp
        image: mustafa199b/demo:nodeapp
        ports:
        - containerPort: 3000
        imagePullPolicy: Always  
---
apiVersion: v1
kind: Service
metadata:
  name: nodeapp
  labels:
    app: nodeapp
spec:
  type: ClusterIP
  selector:
    app: nodeapp
  ports:
  - name: service
    protocol: TCP
    port: 3000
    targetPort: 3000
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: monitoring-node-app
  labels:
    release: monitoring
    app: nodeapp
spec:
  endpoints:
  - path: /metrics
    port: service
    targetPort: 3000
  # The namespaceSelector specifies which namespaces to monitor. In this case, we are monitoring the default namespace.
  namespaceSelector:
    matchNames:
    - default
  selector:
    matchLabels:
      app: nodeapp
```

Apply the deployment and service to the cluster:

```bash
kubectl apply -f k8s-config.yaml
kubectl get pod
kubectl get svc
kubectl get servicemonitor
```

Forward the nodeapp service to a local port to access the app:

```bash
# check browser at localhost:3000 & localhost:3000/metrics
kubectl port-forward svc/nodeapp 3000:3000 &
```

## 16 - Scrape Own Application Metrics & Configure Own Grafana Dashboard (Monitor own App - Part 2) | 15:56

Now let's configure prometheus to scrape the metrics from our own application and create a Grafana dashboard to visualize the metrics.

Apply the servicemonitor resource to the cluster:

```bash
kubectl apply -f k8s-config.yaml
kubectl get servicemonitor
```

After few mins, check the Prometheus UI under "Targets" tab to see if the nodeapp service is being scraped and if the metrics are being collected. You can also check the "Graph" tab to query and visualize the nodeapp metrics.

Create a Grafana dashboard for the nodeapp using the metrics exposed by the Prometheus client library. We can create panels for metrics such as `http_request_operations_total` and `http_request_duration_seconds` and visualize them in a dashboard.

```yaml
# Requests per second
rate(http_request_operations_total[2m])
```

Add another visualization panel for the request duration histogram:

```yaml
# request duration per second
rate(http_request_duration_seconds_sum[2m])
```
