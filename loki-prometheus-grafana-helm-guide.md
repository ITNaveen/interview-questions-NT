# Loki-Prometheus-Grafana Monitoring Stack: Helm Deployment Workflow

## Helm Deployment Process

### 1. Add Required Helm Repositories

```bash
# Add Grafana repository (contains both Loki and Grafana charts)
helm repo add grafana https://grafana.github.io/helm-charts

# Add Prometheus repository 
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Update repositories
helm repo update
```

### 2. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### 3. Deploy Prometheus Stack (includes kube-prometheus-stack)

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=standard \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

The kube-prometheus-stack includes:
- Prometheus server
- Alertmanager
- Node exporter (as DaemonSet)
- kube-state-metrics
- Various ServiceMonitors
- Grafana (which we'll disable and deploy separately)

### 4. Deploy Loki Stack

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.storageClassName=standard \
  --set loki.persistence.size=10Gi
```

The loki-stack includes:
- Loki log aggregation server
- Promtail (as DaemonSet) for log collection

### 5. Deploy Grafana (Separately for better control)

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.storageClassName=standard \
  --set persistence.size=5Gi \
  --set datasources."datasources\.yaml".apiVersion=1 \
  --set datasources."datasources\.yaml".datasources[0].name=Prometheus \
  --set datasources."datasources\.yaml".datasources[0].type=prometheus \
  --set datasources."datasources\.yaml".datasources[0].url=http://prometheus-server.monitoring.svc.cluster.local \
  --set datasources."datasources\.yaml".datasources[0].access=proxy \
  --set datasources."datasources\.yaml".datasources[1].name=Loki \
  --set datasources."datasources\.yaml".datasources[1].type=loki \
  --set datasources."datasources\.yaml".datasources[1].url=http://loki.monitoring.svc.cluster.local:3100 \
  --set datasources."datasources\.yaml".datasources[1].access=proxy
```

### 6. Verify Deployment

```bash
kubectl get pods -n monitoring
```

You should see pods for:
- Prometheus server and operators
- Alertmanager
- Node exporter (one per node)
- kube-state-metrics
- Loki
- Promtail (one per node)
- Grafana

## Data Flow Architecture

### Metrics Collection Workflow (Prometheus) - 

1. **Node Exporter** (DaemonSet):
   - Runs on every Kubernetes node
   - Collects hardware and OS metrics (CPU, memory, disk, network)
   - Exposes metrics at `/metrics` endpoint on port 9100
   - **Industry Practice**: Used as the standard for host-level metrics

2. **kube-state-metrics** (Deployment):
   - Runs as a single deployment
   - Generates metrics about Kubernetes objects (pods, deployments, etc.)
   - Exposes metrics at `/metrics` endpoint
   - **Industry Practice**: Essential for Kubernetes platform monitoring

3. **ServiceMonitors/PodMonitors**:
   - Custom resources created by Prometheus Operator
   - Define which services/pods to scrape
   - Configure scraping intervals and endpoints
   - **Industry Practice**: Used to declaratively define scraping targets

4. **Prometheus Server**:
   - Reads ServiceMonitor/PodMonitor resources
   - Scrapes metrics from defined targets
   - Stores time-series data
   - Provides PromQL query interface
   - **Industry Practice**: Central metrics repository

Node Exporter and Kube-State-Metrics expose metrics endpoints that Prometheus scrapes based on configurations defined in ServiceMonitors and PodMonitors.

# alerting for promethes - 

Update your prometheus-values.yaml file to include:

Slack configuration for AlertManager
Your alert rules (CPU, memory, pod health, etc.)

Run the Helm upgrade command:
bashCopyhelm upgrade prometheus prometheus-community/kube-prometheus-stack -f prometheus-values.yaml -n monitoring

That's it! After the upgrade completes:
Your alerting rules will be active in Prometheus
AlertManager will be configured to send notifications to Slack
When alert conditions are met, you'll receive messages in your Slack channel

# Setting alert - 
1. kubectl get pods -n <namespace> to check prom.. and alertmanager is installed.
2. Prometheus Chart Values: If you used the Prometheus Helm chart, ensure that AlertManager is enabled in the values.yaml file, it should be available.
3. then create prom-value.yaml = helm get values prometheus -n monitoring --all > prometheus-values.yaml.
4. then add slack part in prometheus-values.yaml - 
```yml
alertmanager:                        # Top-level configuration for AlertManager
  config:                            # The actual configuration for AlertManager
    global:                          # Global settings for alert handling
      resolve_timeout: 5m           # How long to wait before marking an alert as resolved

    route:                           # The routing tree of how alerts are handled
      receiver: 'slack-notifications'  # Default receiver name (defined below)
      group_by: ['alertname', 'severity']  # Alerts with same labels are grouped together
      group_wait: 30s               # Wait 30s before sending initial notifications
      group_interval: 5m            # Minimum time between alerts for the same group
      repeat_interval: 1h           # How often to resend ongoing alerts

    receivers:                       # List of receivers to notify
      - name: 'slack-notifications'   # Must match the receiver name above
        slack_configs:              # Slack-specific configuration
          - channel: '#alerts-channel'   # Slack channel to send alerts to
            send_resolved: true     # Send a message when the alert is resolved
            api_url: 'https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ'  
            # Slack webhook URL – use the one you generate from your Slack workspace
```
5. upgrade helm chart = helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring -f prometheus-values.yaml.
6. define prometheus-alerts.yaml = 
```yml
apiVersion: monitoring.coreos.com/v1               # PrometheusRule API version
kind: PrometheusRule                               # Custom resource for Prometheus alert rules
metadata:
  name: infra-alerts                               # Name of the PrometheusRule object
  namespace: monitoring                            # Namespace where Prometheus is installed
  labels:
    release: prometheus                            # Match this to your Helm release name (important!)

spec:
  groups:
    - name: infra.rules                            # Name of the alert group
      rules:

        # ✅ 1. High CPU Usage Alert
        - alert: HighCPUUsage
          expr: avg(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod) > 0.8
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High CPU usage detected"
            description: "Pod {{ $labels.pod }} is using more than 80% CPU for over 2 minutes."

        # ✅ 2. High Memory Usage Alert
        - alert: HighMemoryUsage
          expr: avg(container_memory_usage_bytes{container!=""}) by (pod) > 500000000
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "High Memory usage detected"
            description: "Pod {{ $labels.pod }} is using more than ~500MB RAM for over 2 minutes."
# The High Memory Usage alert is still valuable even when memory limits are set because it gives us an early warning that a pod is approaching its limit before Kubernetes kills and restarts it. Constant restarts can cause performance degradation, downtime, or service interruptions. The alert helps us investigate potential memory leaks, adjust resource requests/limits, or scale pods to avoid frequent pod restarts, ensuring smoother and more stable service operation.

        # ✅ 4. Node Unreachable
        - alert: NodeUnreachable
          expr: up{job="kubelet"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Node unreachable"
            description: "Kubelet is not running or node is down on {{ $labels.instance }}."

        # ✅ 5. Low Disk Space (root filesystem)
        - alert: LowDiskSpace
          expr: (node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}) < 0.15
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Low disk space"
            description: "Node {{ $labels.instance }} has less than 15% disk space left on root."

        # ✅ 6. Kubernetes API Server Errors
        - alert: KubeAPIErrorRate
          expr: rate(apiserver_request_total{code=~"5.."}[5m]) > 1
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "High API error rate"
            description: "Kubernetes API server is returning 5xx errors >1 req/sec."
# What It Does:
This alert is designed to monitor the error rate of requests to the Kubernetes API server (the heart of the Kubernetes control plane). It specifically looks at 5xx HTTP errors (which indicate server-side problems) and checks if the rate of those errors exceeds a given threshold.

HTTP 5xx errors indicate server-side issues such as:
Internal Server Errors (500)
Service Unavailable (503), etc.

If the error rate is too high, it could mean something is seriously wrong with the Kubernetes API, possibly impacting control plane operations, pod scheduling, or cluster health.
```
7. Make sure the release: label matches your Helm release name (often prometheus).
8. kubectl apply -f prometheus-alerts.yaml.

# high memory alert is still needed because - 
```yml
resources:
  requests:
    memory: "500Mi"
  limits:
    memory: "1Gi"
```
Memory Request: Kubernetes will reserve 500 MiB of memory for this pod.
Memory Limit: The pod cannot use more than 1 GiB of memory.
If the pod starts using 1.2 GiB of memory (i.e., exceeds the memory limit):
Kubernetes kills the pod.
The pod is marked as OOMKilled and will restart automatically.
The pod will continue to restart until either:
The pod’s memory usage stays below the limit (e.g., it gets optimized).
The memory limit is adjusted (increased).

# for loki - 
### Log Collection Workflow

1. **Promtail** (DaemonSet):
   - Runs on every Kubernetes node
   - Collects container logs from `/var/log/pods/`
   - Adds Kubernetes metadata as labels (namespace, pod, container)
   - Sends logs directly to Loki
   - **Industry Practice**: Standard log collector for Loki

2. **Loki**:
   - Receives logs from Promtail
   - Indexes logs based on labels (not full text)
   - Stores compressed logs
   - Provides LogQL query interface
   - **Industry Practice**: Efficient log storage designed to work with Grafana

3. **Important**: Logs do NOT go to Prometheus. The data flows are separate:
   - Metrics: Various exporters → Prometheus → Grafana
   - Logs: Containers → Promtail → Loki → Grafana

### Visualization Workflow

1. **Grafana**:
   - Configured with two data sources:
     - Prometheus for metrics
     - Loki for logs
   - Provides dashboards for visualization
   - Allows for combined views of metrics and logs
   - **Industry Practice**: De-facto standard for observability visualization

Promtail: An agent that collects logs from various sources and forwards them to Loki for storage and analysis. ​
infracloud.io

Loki: A log aggregation system that stores logs efficiently, integrating seamlessly with Grafana for visualization.​

Node Exporter: A Prometheus exporter that collects hardware and OS metrics from nodes, such as CPU usage, memory usage, disk I/O, and network statistics. ​
DevOps.dev

Kube-State-Metrics: An add-on service that listens to the Kubernetes API server and generates metrics about the state of Kubernetes objects, including pods, deployments, and services.​

ServiceMonitor: A custom resource definition (CRD) in Kubernetes that defines how Prometheus should discover and scrape metrics from services.​

Prometheus: A monitoring and alerting toolkit that collects and stores metrics data, scraping configured targets like Node Exporter, Kube-State-Metrics, and services defined in ServiceMonitors.​

Grafana: A visualization tool that queries Prometheus for metrics and Loki for logs, providing unified dashboards and insights.

## What You Can See in Grafana

1. **Kubernetes Cluster Dashboards**:
   - Node metrics (CPU, memory, disk, network)
   - Pod resource usage and status
   - Deployment metrics
   - **Industry Practice**: Usually start with pre-built dashboards from kube-prometheus-stack

2. **Application Metrics**:
   - Custom metrics exposed by your applications
   - Request rates, latencies, error rates
   - JVM metrics, database connections, queue depths
   - **Industry Practice**: RED method (Rate, Error, Duration) for services

3. **Log Exploration**:
   - Search logs across all applications
   - Filter by namespace, pod, container
   - Correlate logs with metrics spikes
   - **Industry Practice**: Use LogQL to filter and analyze logs

4. **Combined Views**:
   - Connect logs and metrics in a single dashboard
   - Drill down from metrics anomalies to logs
   - **Industry Practice**: Create context-linked dashboards

## Common Interview Questions and Industry Practices

1. **"How do you monitor Kubernetes clusters?"**
   - "We use the Prometheus-Grafana stack deployed via Helm. Node Exporter collects host metrics, kube-state-metrics provides Kubernetes object metrics, and we visualize everything in Grafana dashboards."

2. **"How do you handle log aggregation?"**
   - "We deploy the Loki stack alongside Prometheus. Promtail, which runs as a DaemonSet on every node, collects container logs and ships them to Loki. We then query and visualize these logs in Grafana, giving us a single pane of glass for both metrics and logs."

3. **"How do you set up alerting?"**
   - "We define PrometheusRule resources for metric-based alerts and configure Loki for log-based alerts. These alerts are routed through Alertmanager to appropriate channels - critical alerts go to PagerDuty for immediate response, while less severe alerts go to Slack for awareness."

4. **"What's your approach to dashboard creation?"**
   - "We start with the pre-built dashboards from kube-prometheus-stack and then create custom dashboards for specific applications. We follow a hierarchical approach - cluster overview, namespace overview, and then specific application dashboards, allowing us to drill down from high-level to detailed views."

5. **"How do you handle persistent storage for monitoring components?"**
- "Since we're fully AWS-based, our persistent storage strategy for monitoring components like Prometheus and Loki is optimized for the cloud environment:
EBS-backed storage: We configure Prometheus and Loki to use AWS EBS volumes through the gp3 storage class. 
In our Helm values, we set up PersistentVolumeClaims with this storage class and appropriate size—typically 100-200GB for Prometheus and similar for Loki in production environments.
Storage configuration: In our Helm charts, we enable persistence with configurations like:
yamlpersistence:
  enabled: true
  storageClass: gp3
  size: 150Gi
  accessMode: ReadWriteOnce

Data lifecycle: We implement retention policies within Prometheus and Loki (e.g., 15 days for raw metrics, 30+ days for aggregated data) to manage storage growth.
Snapshot backups: We leverage AWS EBS snapshots on a scheduled basis using a tool like Velero to protect our monitoring data.
This approach ensures we maintain historical monitoring data even through pod or node failures, which is essential for incident investigation and performance analysis

