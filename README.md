# Lab: OpenTelemetry-Native Infrastructure Monitoring on EKS - in-progress

### Objective

Build a modern infrastructure observability platform using:

```
EKS
 │
 ├── OpenTelemetry Collector (DaemonSet)
 │     ├── Host Metrics
 │     ├── Node Metrics
 │     ├── Pod Metrics
 │     ├── Container Metrics
 │     └── Logs
 │
 ├── Prometheus
 │
 ├── Alertmanager
 │
 └── Grafana
```

#### Key Points

* No Node Exporter
* No OpenTelemetry Operator
* OpenTelemetry Collector acts as the single telemetry agent

***

## Create cluster

```
 eksctl create cluster --name my-cluster --region us-east-1 --version 1.34 --nodegroup-name standard-workers --node-type t3.large --nodes 1 --nodes-min 1 --nodes-max 3
```

## Verify EKS Cluster

```bash
kubectl get nodes

kubectl get all -A
```

Expected:

```
STATUS = Ready
```

Example:

```bash
kubectl get nodes
```

```
NAME                         STATUS   ROLES   AGE
ip-10-0-1-248.ec2.internal   Ready    <none>  15m
```

## Verify EBS CSI Driver

```bash
kubectl get pods -A | grep ebs
```

Expected:

```
ebs-csi-controller
ebs-csi-node
```

***

## Step 3 - Create Monitoring Namespace

```bash
kubectl create namespace monitoring

kubectl get ns
```

***

## Step 4 - Add Helm Repositories

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm repo ls
```

## Create OpenTelemetry Collector Values File

Create file:

```
cat > otel-values.yaml <<'EOF'
mode: daemonset

image:
  repository: otel/opentelemetry-collector-contrib
  tag: 0.163.2

presets:
  hostMetrics:
    enabled: true

  kubeletMetrics:
    enabled: true

  kubernetesAttributes:
    enabled: true

config:
  exporters:
    prometheus:
      endpoint: 0.0.0.0:8889

  service:
    pipelines:
      metrics:
        receivers:
          - hostmetrics
          - kubeletstats
        processors:
          - batch
        exporters:
          - prometheus
EOF
```

## If required we also use for testing otel LB

```
cat > otel-lb.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: otel-collector-lb
  namespace: monitoring
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: opentelemetry-collector
  ports:
  - name: metrics
    port: 8889
    targetPort: 8889
EOF

```

## Install OpenTelemetry Collector

```bash
helm upgrade --install otel-collector \
open-telemetry/opentelemetry-collector \
-n monitoring \
-f otel-values.yaml
```

If LB required

```
kubectl apply -f otel-lb.yaml
```

Verify:

```bash
kubectl get pods -n monitoring

kubectl get ds -n monitoring
```

Expected:

```
DESIRED = Number of Nodes
READY   = Number of Nodes
```

## Verify OTel Metrics Endpoint

Port Forward:

```bash
kubectl port-forward \
svc/otel-collector-opentelemetry-collector \
8889:8889 \
-n monitoring
```

Open:

```
http://localhost:8889/metrics
```

Look for:

```
system_cpu_time
system_memory_usage
system_disk_io
system_network_io
system_filesystem_usage
```

## Install Prometheus

Create:

```bash
vi prometheus-values.yaml
```

Contents:

```yaml
server:
  service:
    type: LoadBalancer

extraScrapeConfigs: |
  - job_name: otel-collector

    static_configs:
      - targets:
        - otel-collector-opentelemetry-collector.monitoring.svc.cluster.local:8889
```

If now pvc required:

```
server:
  service:
    type: LoadBalancer
  persistence:
    enabled: false

alertmanager:
  enabled: false
  persistence:
    enabled: false

extraScrapeConfigs: |
  - job_name: otel-collector
    static_configs:
      - targets:
        - otel-collector-lb.monitoring.svc.cluster.local:8889
```

Install:

```bash
helm upgrade --install prometheus \
prometheus-community/prometheus \
-n monitoring \
-f prometheus-values.yaml
```

Verify:

```bash
kubectl get svc -n monitoring
```

Expected:

```
prometheus-server LoadBalancer
```

## Access Prometheus

```bash
kubectl get svc -n monitoring
```

Example:

```
prometheus-server LoadBalancer
```

Open:

```
http://<PROMETHEUS-LB>
```

Navigate:

```
Status -> Targets
```

Expected:

```
otel-collector = UP
```

## Install Alertmanager

```bash
helm upgrade --install alertmanager \
prometheus-community/alertmanager \
-n monitoring
```

Expose:

```bash
helm upgrade --install alertmanager   prometheus-community/alertmanager   --set persistence.enabled=false   --set service.type=LoadBalancer   -n monitoring

```

Verify:

```bash
kubectl get svc -n monitoring
```

Expected:

```
alertmanager LoadBalancer
```

## Install Grafana

```bash
helm upgrade --install grafana grafana/grafana --set persistence.enabled=false   --set service.type=LoadBalancer   -n monitoring

```

Verify:

```bash
kubectl get svc -n monitoring
```

## Retrieve Grafana Password

```bash
kubectl get secret \
grafana \
-n monitoring \
-o jsonpath="{.data.admin-password}" \
| base64 --decode
```

Username:

```
admin
```

Password:

```
<generated-password>
```

## Configure Grafana Datasource

Login to Grafana.

Add datasource:

```
Prometheus
```

URL:

```
http://prometheus-server.monitoring.svc.cluster.local
```

Click:

```
Save & Test
```

Expected:

```
Data source is working
```

## Verify Infrastructure Metrics

CPU:

```promql
system_cpu_utilization
```

Memory:

```promql
system_memory_utilization
```

Disk:

```promql
system_filesystem_utilization
```

Network:

```promql
system_network_io
```

Pod CPU:

```promql
k8s_pod_cpu_usage
```

## Test High CPU Alert

Create:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: cpu-stress

spec:
  replicas: 4

  selector:
    matchLabels:
      app: cpu-stress

  template:
    metadata:
      labels:
        app: cpu-stress

    spec:
      containers:
      - name: stress
        image: polinux/stress

        command:
        - stress

        args:
        - "--cpu"
        - "2"
```

Deploy:

```bash
kubectl apply -f cpu-stress.yaml
```

Verify:

```bash
kubectl top nodes
```

Expected:

```
CPU utilization increases
```

## Test High Memory Alert

Create:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: memory-stress

spec:
  replicas: 3

  selector:
    matchLabels:
      app: memory-stress

  template:
    metadata:
      labels:
        app: memory-stress

    spec:
      containers:
      - name: stress

        image: polinux/stress

        args:
        - "--vm"
        - "1"
        - "--vm-bytes"
        - "512M"
```

Deploy:

```bash
kubectl apply -f memory-stress.yaml
```

Expected:

```
Memory utilization rises
```

## Test Pod Down Alert

Deploy nginx:

```bash
kubectl create deployment nginx \
--image=nginx
```

Scale to zero:

```bash
kubectl scale deployment nginx \
--replicas=0
```

Expected Alert:

```
PodNotRunning
```

## Test CrashLoopBackOff

Create:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: crash-app

spec:
  replicas: 1

  selector:
    matchLabels:
      app: crash

  template:
    metadata:
      labels:
        app: crash

    spec:
      containers:
      - name: crash

        image: busybox

        command:
        - sh
        - -c
        - "exit 1"
```

Apply:

```bash
kubectl apply -f crash.yaml
```

Expected:

```
CrashLoopBackOff
```

## Create Node CPU Alert

Query:

```promql
avg(system_cpu_utilization) * 100 > 80
```

Condition:

```
For 5 Minutes
```

Action:

```
Email
```

## Create Node Memory Alert

Query:

```promql
avg(system_memory_utilization) * 100 > 85
```

Condition:

```
For 5 Minutes
```

## Create Disk Usage Alert

Query:

```promql
avg(system_filesystem_utilization) * 100 > 90
```

Condition:

```
For 10 Minutes
```

***

## Cleanup

```bash
kubectl delete deployment cpu-stress

kubectl delete deployment memory-stress

kubectl delete deployment nginx

kubectl delete deployment crash-app
```

Optional:

```bash
helm uninstall grafana -n monitoring

helm uninstall alertmanager -n monitoring

helm uninstall prometheus -n monitoring

helm uninstall otel-collector -n monitoring

kubectl delete namespace monitoring
```

## Expected Outcome

* OpenTelemetry Collector DaemonSet
* Host Metrics Collection
* Node CPU Monitoring
* Node Memory Monitoring
* Disk Monitoring
* Filesystem Monitoring
* Network Monitoring
* Pod Monitoring
* Container Monitoring
* Prometheus Metrics Storage
* Alertmanager Notifications
* Grafana Dashboards
* Kubernetes Infrastructure Observability
* OpenTelemetry as Single Telemetry Agent
