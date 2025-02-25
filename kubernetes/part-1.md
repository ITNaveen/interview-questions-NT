# Kubernetes Intermediate Level Q&A Guide

This comprehensive guide contains 15 questions and answers covering intermediate-level Kubernetes concepts with clear examples and explanations.

## 1. How do you implement rolling updates in Kubernetes, and what are the key parameters?

**Answer:** Rolling updates in Kubernetes allow you to update your application without downtime by gradually replacing old pods with new ones.

The key parameters for rolling updates in a Deployment are:
- `maxUnavailable`: Maximum number of pods that can be unavailable during the update
- `maxSurge`: Maximum number of pods that can be created over the desired number of pods

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
```

In this deployment, during an update:
- At most 1 pod can be unavailable (`maxUnavailable: 1`)
- At most 1 pod can be created over the desired count (`maxSurge: 1`)

To update the image:
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.20
```

The update will proceed gradually, ensuring that at least 4 pods (out of 5) are always available, and never creating more than 6 pods total.

## 2. Explain the concept of StatefulSets and how they differ from Deployments.

**Answer:** StatefulSets are workload resources designed for stateful applications that require stable, unique network identifiers, persistent storage, and ordered deployment and scaling.

**Key differences from Deployments:**

| Feature | StatefulSet | Deployment |
|---------|------------|------------|
| Pod Names | Predictable, persistent identifiers (e.g., app-0, app-1) | Random names with hash (e.g., app-5d87f58c9b) |
| Pod Creation/Deletion | Sequential (ordered) | Parallel (unordered) |
| Scaling | Sequential | Parallel |
| Volume Handling | Stable persistent volume claims per pod | Shared volumes across pods |
| DNS Names | Stable, predictable hostname per pod | Random hostnames |

**Example of a StatefulSet:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.19
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

This will create three pods named `web-0`, `web-1`, and `web-2`, each with its own persistent volume claim.

## 3. What are Kubernetes Operators and when would you use them?

**Answer:** Kubernetes Operators are application-specific controllers that extend Kubernetes functionality to manage complex, stateful applications using custom resources. They encapsulate operational knowledge (like upgrades, backups, or failovers) that would typically require manual intervention.

**When to use Operators:**
- For complex stateful applications (databases, monitoring systems, etc.)
- When applications require specific domain knowledge for operations
- To automate day-2 operations like backups, scaling, and upgrades
- When standard Kubernetes primitives aren't sufficient

**Example - Creating a simple Operator using the Operator SDK:**

1. Define a Custom Resource Definition (CRD):
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqlinstances.example.com
spec:
  group: example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                replicas:
                  type: integer
                version:
                  type: string
  scope: Namespaced
  names:
    plural: mysqlinstances
    singular: mysqlinstance
    kind: MySQLInstance
    shortNames:
    - msql
```

2. Create the custom resource:
```yaml
apiVersion: example.com/v1
kind: MySQLInstance
metadata:
  name: mysql-sample
spec:
  replicas: 3
  version: "8.0"
```

3. The operator would watch for these custom resources and handle all the complex deployment, scaling, and management of the MySQL instances.

Popular examples include the Prometheus Operator, Elasticsearch Operator, and PostgreSQL Operator.

## 4. How does Horizontal Pod Autoscaling work in Kubernetes? Can you scale based on custom metrics?

**Answer:** Horizontal Pod Autoscaling (HPA) automatically scales the number of pods in a deployment or replicaset based on observed metrics.

**How it works:**
1. The HPA controller periodically checks metrics from the Metrics API
2. It calculates the desired number of replicas based on current vs. target metrics
3. It updates the replicas field of the target resource (Deployment/ReplicaSet/etc.)

By default, it can scale based on CPU utilization, but it supports custom metrics through the custom metrics API.

**Example - CPU-based autoscaling:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Example - Custom metrics-based autoscaling:**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1000
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1
        kind: Ingress
        name: main-route
      target:
        type: Value
        value: 2000
```

To use custom metrics, you need to set up a metrics solution like Prometheus and the Prometheus Adapter.

## 5. Explain Kubernetes NetworkPolicies and provide an example of how to restrict pod communication.

**Answer:** NetworkPolicies are Kubernetes resources that control the traffic flow between pods using policy rules. They act like a firewall, specifying which pods can communicate with each other and with external endpoints.

By default (without NetworkPolicies), all pods can communicate with any other pod. With NetworkPolicies, you can enforce restrictions like:
- Only allowing specific pods to access a database
- Restricting traffic to a specific namespace
- Blocking egress traffic to external endpoints

**Example - Restricting access to a database pod:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-netpolicy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
      namespaceSelector:
        matchLabels:
          purpose: production
    ports:
    - protocol: TCP
      port: 5432
```

This NetworkPolicy:
1. Applies to pods with label `app: database` in the `production` namespace
2. Allows ingress traffic only from pods with label `app: backend` in namespaces labeled `purpose: production`
3. Only allows TCP traffic on port 5432 (PostgreSQL)
4. Implicitly denies all other ingress traffic

NetworkPolicies require a CNI (Container Network Interface) plugin that supports them, such as Calico, Cilium, or Weave Net.

## 6. What are Kubernetes Admission Controllers and how would you use them to enforce security policies?

**Answer:** Admission Controllers are plugins that intercept requests to the Kubernetes API server before persistence but after authentication and authorization. They can modify or reject requests based on custom logic.

There are two types:
- **Validating Admission Controllers**: Only validate requests (can reject but not modify)
- **Mutating Admission Controllers**: Can modify requests before processing

**Common use cases:**
- Enforcing security policies
- Resource quota validation
- Default resource limits
- Pod security standards enforcement
- Image policy validation

**Example - PodSecurityPolicy (deprecated but illustrative):**
```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: true
```

**Example - Using OPA Gatekeeper (modern approach):**
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    labels: ["team"]
    message: "All pods must have a 'team' label"
```

To implement, you would use a validating webhook configuration:
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: policy-validator
webhooks:
- name: policy.example.com
  clientConfig:
    service:
      name: policy-validator
      namespace: security
      path: "/validate"
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
```

## 7. How do you manage secrets securely in a Kubernetes environment?

**Answer:** Kubernetes Secrets are objects that store sensitive information like passwords, tokens, or keys. While convenient, built-in Secrets have some security limitations.

**Best practices for managing Kubernetes secrets:**

1. **Use external secrets management:**
   - HashiCorp Vault
   - AWS Secrets Manager
   - Azure Key Vault
   - Google Secret Manager

2. **Encrypt etcd storage:**
```yaml
# In kube-apiserver configuration
--encryption-provider-config=/etc/kubernetes/encryption/config.yaml
```

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-key>
      - identity: {}
```

3. **Use RBAC to limit access to secrets:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  resourceNames: ["app-secret"]
```

4. **Use sealed secrets:**
```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Create a sealed secret
kubeseal -o yaml < secret.yaml > sealed-secret.yaml
```

5. **Implement pod security contexts:**
```yaml
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
```

6. **Use Secret immutability:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-credentials
  annotations:
    kubectl.kubernetes.io/immutable: "true"
```

**Example - Using external secrets (with External Secrets Operator):**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: database-credentials
  data:
  - secretKey: username
    remoteRef:
      key: database/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: database/credentials
      property: password
```

## 8. Explain Kubernetes Service Mesh concepts and how solutions like Istio enhance Kubernetes networking.

**Answer:** A Service Mesh is a dedicated infrastructure layer for handling service-to-service communication in microservices architectures, offering features beyond what native Kubernetes provides.

**Key Service Mesh concepts:**
- **Sidecar proxies**: Deployed alongside application containers to intercept traffic
- **Control plane**: Manages and configures the proxies
- **Data plane**: Handles the actual traffic flow between services

**Benefits of Service Mesh solutions like Istio:**

1. **Advanced traffic management:**
   - Traffic splitting/canary deployments
   - A/B testing
   - Circuit breaking
   - Fault injection

2. **Security:**
   - Mutual TLS (mTLS) between services
   - Fine-grained access control
   - Certificate management

3. **Observability:**
   - Distributed tracing
   - Metrics collection
   - Visualization

**Example - Implementing canary deployment with Istio:**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Example - Enforcing mTLS with Istio:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

Other popular service mesh implementations include Linkerd, Consul Connect, and AWS App Mesh.

## 9. How would you design a Kubernetes multi-cluster architecture and implement workload federation?

**Answer:** A Kubernetes multi-cluster architecture involves running multiple Kubernetes clusters with some form of workload distribution or federation between them.

**Main approaches to multi-cluster architecture:**

1. **Cluster Federation**
   - Central control plane manages multiple clusters
   - Kubernetes Federation v2 (KubeFed)
   - Allows deploying workloads across multiple clusters

2. **Service Mesh Federation**
   - Using Istio or similar to connect services across clusters
   - Provides service discovery and secure communication

3. **Multi-cluster Control Planes**
   - Tools like Rancher, Fleet, or Anthos
   - Unified management of multiple clusters

**Benefits of multi-cluster:**
- High availability across regions/zones
- Regulatory compliance (data locality)
- Separation of concerns (prod/dev/test)
- Avoiding vendor lock-in
- Scalability beyond single cluster limits

**Example - KubeFed configuration:**
```yaml
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: test-deployment
  namespace: test-namespace
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - image: nginx
            name: nginx
  placement:
    clusters:
    - name: cluster1
    - name: cluster2
  overrides:
  - clusterName: cluster2
    clusterOverrides:
    - path: "/spec/replicas"
      value: 5
```

**Example - Istio multi-cluster setup:**
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: eastwest-gateway
spec:
  profile: empty
  components:
    ingressGateways:
    - name: istio-eastwestgateway
      label:
        app: istio-eastwestgateway
        istio: eastwestgateway
      enabled: true
      k8s:
        env:
        - name: ISTIO_META_ROUTER_MODE
          value: "sni-dnat"
        service:
          ports:
          - name: status-port
            port: 15021
          - name: tls
            port: 15443
            targetPort: 15443
          - name: tls-istiod
            port: 15012
            targetPort: 15012
          - name: tls-webhook
            port: 15017
            targetPort: 15017
```

## 10. Explain how to implement GitOps with tools like ArgoCD or Flux in a Kubernetes environment.

**Answer:** GitOps is an operational framework that takes DevOps best practices for application deployment and applies them to infrastructure automation. In Kubernetes, this means using Git as the single source of truth for declarative infrastructure and applications.

**Key principles of GitOps:**
- Declarative configuration stored in Git
- Configuration can be applied automatically
- System detects and corrects drift from the desired state
- Git serves as audit log and allows rollbacks

**Popular GitOps tools:**
- ArgoCD
- Flux
- Jenkins X
- Rancher Fleet

**Example - Setting up ArgoCD:**

1. Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

2. Create an Application resource:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp.git
    targetRevision: HEAD
    path: kubernetes/manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

This configuration:
- Connects to the Git repo containing Kubernetes manifests
- Deploys resources to the specified namespace
- Automatically syncs when changes are detected in Git
- Prunes resources that are removed from Git
- Self-heals if someone makes manual changes

**Example - Setting up Flux v2:**

1. Install Flux CLI and bootstrap:
```bash
flux bootstrap github \
  --owner=myorg \
  --repository=k8s-infrastructure \
  --branch=main \
  --path=clusters/production \
  --personal
```

2. Create a GitRepository resource:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/myorg/myapp
  ref:
    branch: main
```

3. Create a Kustomization resource:
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 10m
  path: "./kubernetes"
  prune: true
  sourceRef:
    kind: GitRepository
    name: myapp
  targetNamespace: myapp
```

## 11. How do you implement effective resource management and quota enforcement in Kubernetes?

**Answer:** Effective resource management in Kubernetes involves setting appropriate requests and limits, implementing resource quotas, and establishing limit ranges to ensure fair resource allocation and prevent resource starvation.

**Key resource management components:**

1. **Resource Requests and Limits**
   - **Requests**: Minimum resources guaranteed to the pod
   - **Limits**: Maximum resources a pod can use

2. **ResourceQuota**
   - Constrains aggregate resource consumption per namespace
   - Controls number of objects by type

3. **LimitRange**
   - Enforces minimum/maximum resource usage per pod/container
   - Sets default requests/limits if not specified

**Example - Setting resource requests and limits:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.com/app:v4
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

**Example - Implementing a ResourceQuota:**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "10"
    persistentvolumeclaims: "5"
    services: "10"
    configmaps: "10"
    secrets: "10"
```

**Example - Setting a LimitRange:**
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-a
spec:
  limits:
  - default:
      memory: 256Mi
      cpu: 500m
    defaultRequest:
      memory: 128Mi
      cpu: 200m
    max:
      memory: 1Gi
      cpu: "1"
    min:
      memory: 64Mi
      cpu: 100m
    type: Container
```

**Best practices:**
- Set appropriate requests based on actual application needs
- Always specify both requests and limits
- Use namespace quotas to partition cluster resources
- Implement monitoring and alerts for resource usage
- Use autoscaling for dynamic workloads
- Consider Quality of Service (QoS) classes:
  - **Guaranteed**: requests = limits
  - **Burstable**: requests < limits
  - **BestEffort**: no requests or limits

## 12. Explain Custom Resource Definitions (CRDs) and how to create controllers for them.

**Answer:** Custom Resource Definitions (CRDs) extend the Kubernetes API by defining custom resources that allow you to store and retrieve structured data. Controllers for these CRDs implement the business logic to manage these resources.

**CRD components:**
- **Custom Resource Definition**: Defines the schema for the custom resource
- **Custom Resource**: Instance of the defined CRD
- **Custom Controller**: Watches for changes to CRs and takes actions

**Example - Creating a CRD for a Database resource:**
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["engine", "version", "storage"]
            properties:
              engine:
                type: string
                enum: ["mysql", "postgresql"]
              version:
                type: string
              storage:
                type: string
              replicas:
                type: integer
                minimum: 1
                default: 1
          status:
            type: object
            properties:
              phase:
                type: string
              url:
                type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Status
      type: string
      jsonPath: .status.phase
```

**Example - Creating a Custom Resource:**
```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: production-db
spec:
  engine: postgresql
  version: "13.4"
  storage: "10Gi"
  replicas: 3
```

**Example - Building a controller (Go code excerpt):**
```go
// Main reconciliation loop
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Fetch the Database instance
    var db examplev1.Database
    if err := r.Get(ctx, req.NamespacedName, &db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    
    // Check if the StatefulSet already exists, if not create a new one
    var sts appsv1.StatefulSet
    err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, &sts)
    if err != nil && errors.IsNotFound(err) {
        // Define a new StatefulSet
        sts := r.statefulSetForDatabase(&db)
        if err = r.Create(ctx, sts); err != nil {
            return ctrl.Result{}, err
        }
        
        // StatefulSet created, update status
        db.Status.Phase = "Creating"
        if err := r.Status().Update(ctx, &db); err != nil {
            return ctrl.Result{}, err
        }
        
        return ctrl.Result{Requeue: true}, nil
    } else if err != nil {
        return ctrl.Result{}, err
    }
    
    // Update Database status based on StatefulSet status
    if sts.Status.ReadyReplicas == *sts.Spec.Replicas {
        db.Status.Phase = "Running"
        db.Status.URL = fmt.Sprintf("%s.%s.svc.cluster.local", db.Name, db.Namespace)
    } else {
        db.Status.Phase = "Pending"
    }
    
    if err := r.Status().Update(ctx, &db); err != nil {
        return ctrl.Result{}, err
    }
    
    return ctrl.Result{}, nil
}
```

The Operator SDK and Kubebuilder are popular tools for building Kubernetes controllers.

## 13. How would you implement Kubernetes monitoring and observability at scale?

**Answer:** Implementing effective monitoring and observability in a Kubernetes environment requires a multi-layered approach focusing on infrastructure, Kubernetes components, and application metrics.

**Key components of a Kubernetes monitoring stack:**

1. **Metrics collection and visualization:**
   - Prometheus for metrics collection
   - Grafana for dashboards and visualization
   - kube-state-metrics for Kubernetes object metrics
   - node-exporter for node-level metrics

2. **Logging:**
   - Elasticsearch, Fluentd/Fluent Bit, and Kibana (EFK stack)
   - Loki for log aggregation
   - Vector for log processing

3. **Distributed tracing:**
   - Jaeger or Zipkin
   - OpenTelemetry for instrumentation

4. **Alerting:**
   - Alertmanager for alert routing
   - PagerDuty, Slack, or email for notifications

**Example - Setting up Prometheus with Helm:**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

**Example - Prometheus ServiceMonitor (for discovering services to monitor):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: metrics
    interval: 15s
  namespaceSelector:
    matchNames:
    - default
```

**Example - PrometheusRule (for alerting):**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: example-alert
  namespace: monitoring
spec:
  groups:
  - name: example
    rules:
    - alert: HighErrorRate
      expr: sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: High error rate detected
        description: Error rate is above 10% for 10 minutes.
```

**Example - Logging with Fluentd and Elasticsearch:**
```yaml
# Simplified Fluentd ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
  namespace: logging
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>
    
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch
      port 9200
      logstash_format true
      logstash_prefix k8s
    </match>
```

**Best practices for Kubernetes monitoring at scale:**
- Use the Prometheus Operator for managing Prometheus instances
- Implement horizontal scaling for monitoring components
- Set up appropriate retention policies and storage
- Use federation for larger clusters
- Implement proper resource requests/limits for monitoring components
- Focus on golden signals: latency, traffic, errors, and saturation
- Implement SLOs (Service Level Objectives) based on user experience
- Use recording rules to pre-compute expensive queries

