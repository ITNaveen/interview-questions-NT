# Loki will run as a centralized log storage backend, receiving logs from the log collectors. You can deploy Loki as a StatefulSet or Deployment. The following is a simple way to deploy Loki with the official Helm chart: port = 3100
```yml
# Add the Loki Helm repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create a namespace for Loki
kubectl create namespace loki

# Install Loki with the Loki stack, which includes Loki and its components like Loki itself and some helper tools
helm install loki grafana/loki-stack --namespace loki
```

# Now, to collect logs from all Kubernetes nodes and send them to Loki, you need to deploy a log collector like Fluentd or Fluent Bit as a DaemonSet. Both of these tools are capable of collecting logs from containers and forwarding them to Loki.
Example: Deploy Fluent Bit (lightweight log forwarder) with Loki output
You can deploy Fluent Bit as a DaemonSet to collect logs from your containers and forward them to Loki.
```yml
# Create a namespace for Fluent Bit (optional)
kubectl create namespace logging

# Install Fluent Bit using Helm (in the logging namespace)
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit --namespace logging \
  --set "outputs.loki.enabled=true" \
  --set "outputs.loki.url=http://loki:3100/loki/api/v1/push" \
  --set "inputs.tail.enabled=true" \
  --set "inputs.tail.path=/var/log/containers/*.log" \
  --set "inputs.tail.parser=docker"
```
This will deploy Fluent Bit as a DaemonSet on all nodes in your Kubernetes cluster, collect logs from container logs in /var/log/containers/, and send them to your Loki instance. It configures the loki output plugin to send logs to the Loki API endpoint.

# verify - 
```yml
kubectl get pods --namespace loki
kubectl get pods --namespace logging
```
To view your logs in Grafana, you’ll need to configure Grafana to use Loki as a data source.
Go to Grafana UI.
Navigate to Configuration > Data Sources > Loki.
Set the URL to the Loki service you’ve deployed, typically http://loki:3100 if using the default Helm chart settings.
Save and test the connection.

........
,.......
# logs - 
```yml
1. All Logs from a Container - {container="nginx"}
Displays all logs from the nginx container.

2. Logs Containing a Specific Text - {container="nginx"} |= "error"
Shows logs from the nginx container containing the word error.

3. Count of Logs Over Time (e.g., Last 1 Hour) - count_over_time({container="nginx", level="error"}[1h])
Counts error logs from nginx in the last 1 hour.

4. Logs from Specific Pod (e.g., nginx-pod) - {pod="nginx-pod"}
Shows all logs from the nginx-pod.

5. Show Logs for a Specific Namespace - {namespace="default"}
Displays logs from the default namespace.

6. Calculate Latency by Matching Start and End Logs - 
{container="nginx"} |= "request_start" or |= "request_end"
This would show both request_start and request_end logs. You can correlate the start and end logs manually or with external tools to calculate latency.
```




