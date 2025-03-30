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

### Metrics Collection Workflow

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

## Alerting Configuration

### Setting Up Prometheus Alerts

1. **PrometheusRule Custom Resource**:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: app-alerts
  namespace: monitoring
  labels:
    release: prometheus  # Important: match your Helm release label
spec:
  groups:
  - name: kubernetes
    rules:
    - alert: HighCPUUsage
      expr: sum(rate(container_cpu_usage_seconds_total{container!="POD",container!=""}[5m])) by (pod) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "Pod {{ $labels.pod }} has high CPU usage ({{ $value }})"
```

2. **Apply the rules**:

```bash
kubectl apply -f prometheus-rules.yaml
```

3. **Industry Practice**: Group alerts by service or component, use consistent severity labels

### Setting Up Loki Alerts (via Grafana)

1. **Create Loki alert in Grafana UI**:
   - Navigate to Alerting > Alert Rules
   - Create a new alert rule using LogQL:
     - Example: `sum(rate({app="my-app"} |= "error" [5m])) > 10`
   - Set evaluation interval and alerting thresholds
   - Configure notification channels

2. **Industry Practice**: 
   - Use Loki for pattern-based log alerts
   - Use Prometheus for metric-based alerts

### Configuring Alertmanager

1. **Create a values file for AlertManager configuration**:

```yaml
# alertmanager-values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['job', 'alertname', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'slack'
      routes:
      - match:
          severity: critical
        receiver: 'pagerduty'
    receivers:
    - name: 'slack'
      slack_configs:
      - api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXX'
        channel: '#alerts'
        send_resolved: true
    - name: 'pagerduty'
      pagerduty_configs:
      - service_key: 'your_pagerduty_service_key'
```

2. **Update Prometheus stack with AlertManager config**:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f alertmanager-values.yaml
```

3. **Industry Practice**:
   - Route different severity alerts to different channels
   - Critical alerts to PagerDuty/OpsGenie for immediate action
   - Warning alerts to Slack/Teams for awareness
   - Group related alerts to reduce notification noise

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
   - "We configure persistent storage for Prometheus and Loki through their Helm values, using the appropriate storage class for our environment. This ensures we retain historical metrics and logs even if pods restart."

6. **"How do you scale your monitoring stack?"**
   - "For larger clusters, we increase resource limits for Prometheus and Loki, implement retention policies to manage storage, and in some cases shard Prometheus by namespace or application to distribute the load."
