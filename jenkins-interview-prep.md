# Jenkins Interview Questions - Intermediate to Expert Level

## Question 1: Explain the difference between Declarative and Scripted Pipelines in Jenkins.

**Answer:** 
Declarative and Scripted Pipelines represent two different syntaxes for defining Jenkins pipelines:

**Declarative Pipeline:**
- Provides a more structured and simplified syntax
- Starts with `pipeline` block and uses predefined sections like `agent`, `stages`, `steps`
- Enforces stricter syntax with specific schema
- Easier to read and maintain for those less familiar with Groovy
- Better integration with Blue Ocean Pipeline Editor
- Example:
```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

**Scripted Pipeline:**
- Provides more flexibility and programmability 
- Uses Groovy DSL for more complex logic
- Starts with `node` block
- Offers greater control over execution flow
- Requires better understanding of Groovy
- Example:
```groovy
node {
    stage('Build') {
        echo 'Building...'
    }
}
```

The key difference is that Declarative offers a simpler but more rigid structure, while Scripted provides more flexibility through full access to the Groovy programming language.

## Question 2: What is a Jenkins Shared Library and how would you implement one?

**Answer:**
Jenkins Shared Libraries are reusable code repositories that can be shared across multiple Jenkins pipelines. They help standardize pipeline code, promote best practices, and reduce duplication.

**Implementation steps:**
1. **Create a repository** with a specific structure:
   - `src/` directory for Java classes
   - `vars/` directory for global variables/functions
   - `resources/` directory for non-Groovy files
   
2. **Configure the library in Jenkins**:
   - Go to "Manage Jenkins" > "Configure System"
   - Under "Global Pipeline Libraries" section, add a new library
   - Specify name, default version, and retrieval method (SCM)

3. **Create pipeline functions** in the `vars/` directory:
   ```groovy
   // vars/sayHello.groovy
   def call(String name = 'human') {
       echo "Hello, ${name}!"
   }
   ```

4. **Use the library in your pipeline**:
   ```groovy
   @Library('my-shared-library') _
   
   pipeline {
       agent any
       stages {
           stage('Example') {
               steps {
                   sayHello('Jenkins Expert')
               }
           }
       }
   }
   ```

Shared libraries can contain complex functions like deployment procedures, notification systems, or security scans that can be maintained centrally but used across many pipelines.

## Question 3: How would you implement Infrastructure as Code (IaC) using Jenkins?

**Answer:**
Implementing Infrastructure as Code with Jenkins involves:

1. **Tool integration:**
   - Integrate infrastructure tools like Terraform, CloudFormation, Ansible, or Kubernetes manifests
   - Store configuration files in source control (GitOps approach)

2. **Pipeline setup:**
   ```groovy
   pipeline {
       agent any
       stages {
           stage('Checkout') {
               steps {
                   checkout scm
               }
           }
           stage('Terraform Plan') {
               steps {
                   sh 'terraform init'
                   sh 'terraform plan -out=tfplan'
               }
           }
           stage('Approval') {
               steps {
                   input message: 'Apply the terraform plan?'
               }
           }
           stage('Terraform Apply') {
               steps {
                   sh 'terraform apply -auto-approve tfplan'
               }
           }
       }
   }
   ```

3. **Security considerations:**
   - Manage credentials securely using Jenkins Credentials Plugin
   - Implement principle of least privilege for service accounts

4. **Testing infrastructure:**
   - Add validation stages using tools like Inspec, Terratest, or Goss
   - Implement smoke tests after deployment

5. **State management:**
   - Configure backends for state storage (S3, Azure Blob, etc.)
   - Implement state locking mechanisms to prevent conflicts

This approach ensures infrastructure changes are versioned, tested, reviewed, and applied consistently.

## Question 4: Explain how you would implement a blue-green deployment strategy using Jenkins.

**Answer:**
Blue-green deployment is a technique that reduces downtime by running two identical production environments (blue and green). At any time, only one environment is live.

**Implementation in Jenkins:**

1. **Pipeline structure:**
   ```groovy
   pipeline {
       agent any
       environment {
           BLUE_ENV = 'production-blue'
           GREEN_ENV = 'production-green'
           LIVE_ENV = sh(script: 'kubectl get service main-service -o jsonpath="{.spec.selector.env}"', returnStdout: true).trim()
       }
       stages {
           stage('Determine Deploy Environment') {
               steps {
                   script {
                       if (env.LIVE_ENV == env.BLUE_ENV) {
                           env.DEPLOY_ENV = env.GREEN_ENV
                       } else {
                           env.DEPLOY_ENV = env.BLUE_ENV
                       }
                       echo "Current live: ${env.LIVE_ENV}, Deploying to: ${env.DEPLOY_ENV}"
                   }
               }
           }
           stage('Deploy to Target Environment') {
               steps {
                   sh "kubectl apply -f k8s/${env.DEPLOY_ENV}-deployment.yaml"
                   sh "kubectl rollout status deployment/${env.DEPLOY_ENV}"
               }
           }
           stage('Run Tests Against New Environment') {
               steps {
                   sh "run-integration-tests.sh http://${env.DEPLOY_ENV}"
               }
           }
           stage('Switch Traffic') {
               steps {
                   input message: "Switch traffic to ${env.DEPLOY_ENV}?"
                   sh "kubectl patch service main-service -p '{\"spec\":{\"selector\":{\"env\":\"${env.DEPLOY_ENV}\"}}}'"
               }
           }
           stage('Verify New Environment') {
               steps {
                   sh "verify-deployment.sh"
               }
           }
       }
   }
   ```

2. **Key considerations:**
   - Database migrations need careful handling to maintain compatibility
   - Implement health checks to verify the new environment before switching
   - Create automated rollback procedures if verification fails
   - Use dedicated service/ingress objects to control traffic routing
   - Add monitoring to detect issues quickly after switching

This strategy allows for seamless deployments with minimal downtime and easy rollbacks.

## Question 5: How would you design a Jenkins setup for high availability and disaster recovery?

**Answer:**
A high availability and disaster recovery strategy for Jenkins includes:

1. **Master redundancy:**
   - Use Jenkins Active-Standby setup with a warm standby master
   - Configure automated failover using tools like Keepalived or cloud provider load balancers
   - Implement health checks to detect master failures

2. **State persistence:**
   - Store Jenkins home directory on network storage (NFS, EFS, etc.)
   - Use a database backend for job history instead of the default file system
   - Regularly backup configuration (using plugins like ThinBackup or Configuration as Code)

3. **Distributed builds:**
   - Deploy agents across multiple availability zones/regions
   - Use dynamic agent provisioning (Jenkins Cloud Plugins)
   - Implement labels and node properties for intelligent workload distribution

4. **Infrastructure as Code (IaC):**
   - Define Jenkins installation with tools like Terraform, Ansible, or CloudFormation
   - Use Jenkins Configuration as Code (JCasC) to store all configurations in source control
   - Automate Jenkins setup for quick recovery

5. **Backup strategy:**
   ```
   - Jenkins home directory: Daily backups with incremental strategy
   - Configuration files: Version-controlled and backed up separately
   - Plugins and their versions: Tracked in code
   - Credentials: Stored in external secrets management systems like HashiCorp Vault
   ```

6. **Recovery procedures:**
   - Document and regularly test disaster recovery procedures
   - Create automated recovery scripts
   - Define RTO (Recovery Time Objective) and RPO (Recovery Point Objective)
   - Implement monitoring and alerting for early detection of issues

7. **Testing:**
   - Regularly simulate disaster scenarios to verify recovery procedures
   - Conduct fire drills to ensure team familiarity with recovery processes

This comprehensive approach ensures Jenkins remains available and recoverable even in the event of significant infrastructure failures.

## Question 6: Explain how you would implement security scanning in Jenkins pipelines.

**Answer:**
Implementing security scanning in Jenkins pipelines involves several components:

1. **SAST (Static Application Security Testing):**
   ```groovy
   stage('SAST') {
       steps {
           // SonarQube analysis
           withSonarQubeEnv('SonarQube') {
               sh "mvn sonar:sonar"
           }
           
           // Wait for quality gate
           timeout(time: 10, unit: 'MINUTES') {
               waitForQualityGate abortPipeline: true
           }
           
           // Checkmarx scanning
           checkmarx(
               projectName: 'MyProject',
               teamPath: 'CxServer\\SP\\Company',
               incremental: true,
               excludeFolders: 'tests,node_modules',
               waitForResultsEnabled: true
           )
       }
   }
   ```

2. **Dependency scanning:**
   ```groovy
   stage('Dependency Check') {
       steps {
           // OWASP dependency check
           dependencyCheck(
               additionalArguments: '--scan ./ --suppression suppression.xml --failOnCVSS 7',
               odcInstallation: 'OWASP-DC'
           )
           
           // Snyk scanning
           snykSecurity(
               snykInstallation: 'Snyk',
               snykTokenId: 'snyk-api-token',
               failOnIssues: true
           )
       }
   }
   ```

3. **Container security scanning:**
   ```groovy
   stage('Container Security') {
       steps {
           // Trivy container scanning
           sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
           
           // Anchore Engine scan
           anchore name: IMAGE_NAME, bailOnFail: true
       }
   }
   ```

4. **DAST (Dynamic Application Security Testing):**
   ```groovy
   stage('DAST') {
       steps {
           // OWASP ZAP scanning
           sh "docker run -t owasp/zap2docker-stable zap-baseline.py -t https://my-application-staging.example.com -r zap-report.html"
           publishHTML([
               allowMissing: false,
               alwaysLinkToLastBuild: true,
               reportDir: './',
               reportFiles: 'zap-report.html',
               reportName: 'ZAP Security Report'
           ])
       }
   }
   ```

5. **Infrastructure scanning:**
   ```groovy
   stage('Infrastructure Scan') {
       steps {
           // Terraform security scanning
           sh "tfsec ."
           
           // CloudFormation scanning
           sh "cfn-nag *.yaml"
       }
   }
   ```

6. **Secret scanning:**
   ```groovy
   stage('Secret Scan') {
       steps {
           sh "trufflehog --regex --entropy=True ."
       }
   }
   ```

7. **Compliance checks:**
   ```groovy
   stage('Compliance') {
       steps {
           // InSpec for compliance scanning
           sh "inspec exec compliance-profile -t ssh://user@host --reporter html:inspec-report.html"
           publishHTML([
               allowMissing: false,
               alwaysLinkToLastBuild: true,
               reportDir: './',
               reportFiles: 'inspec-report.html',
               reportName: 'Compliance Report'
           ])
       }
   }
   ```

8. **Integration with security gates:**
   - Define quality gates based on security metrics
   - Configure automatic build failure for critical vulnerabilities
   - Implement notifications to security teams

This comprehensive approach ensures applications are scanned for vulnerabilities at multiple levels throughout the delivery pipeline.

## Question 7: How would you optimize Jenkins build performance and pipeline efficiency?

**Answer:**
Optimizing Jenkins build performance involves several strategies:

1. **Pipeline optimizations:**
   - **Parallel execution:** Run independent stages concurrently
     ```groovy
     parallel {
         stage('Unit Tests') {
             steps { sh './run-unit-tests.sh' }
         }
         stage('Static Analysis') {
             steps { sh './run-static-analysis.sh' }
         }
     }
     ```
   - **Skip stages conditionally:** Use `when` directives to avoid unnecessary work
     ```groovy
     when {
         expression { return env.BRANCH_NAME != 'main' }
     }
     ```
   - **Incremental builds:** Only rebuild what changed

2. **Agent optimizations:**
   - **Agent sizing:** Allocate appropriate resources (CPU/RAM) for build types
   - **Agent pools:** Create specialized agents for specific workloads
   - **Dynamically provisioned agents:** Scale with demand using cloud plugins
     ```groovy
     agent {
         kubernetes {
             yaml """
             apiVersion: v1
             kind: Pod
             spec:
               containers:
               - name: maven
                 image: maven:3.8.4-openjdk-11
                 command: ['cat']
                 tty: true
                 resources:
                   requests:
                     memory: "2Gi"
                     cpu: "500m"
                   limits:
                     memory: "4Gi"
                     cpu: "1000m"
             """
         }
     }
     ```

3. **Caching strategies:**
   - **Artifact caching:** Use Nexus, Artifactory, or similar tools
   - **Docker layer caching:** Leverage build cache in multi-stage Dockerfiles
   - **Dependency caching:** Implement Maven/Gradle/npm caches
     ```groovy
     sh "mvn -B clean verify -Dmaven.repo.local=${WORKSPACE}/.m2/repository"
     ```

4. **Resource management:**
   - **Workspace cleanup:** Regularly clean workspaces to free disk space
     ```groovy
     cleanWs()
     ```
   - **Disk I/O:** Use RAM disks for I/O-intensive operations
   - **Timeout limits:** Set reasonable timeouts to avoid hung builds
     ```groovy
     timeout(time: 1, unit: 'HOURS')
     ```

5. **Process improvements:**
   - **Pipeline templates:** Standardize common patterns
   - **Shared libraries:** Extract common functionality
   - **Reduce build size:** Use sparse checkouts, exclude unnecessary files
     ```groovy
     checkout([
         $class: 'GitSCM',
         extensions: [
             [$class: 'SparseCheckoutPaths', sparseCheckoutPaths: [
                 [$class: 'SparseCheckoutPath', path: 'path/to/include/']
             ]]
         ]
     ])
     ```

6. **Infrastructure tuning:**
   - **JVM tuning:** Optimize Jenkins master JVM settings
     ```
     JAVA_OPTS="-Xmx4g -Xms2g -XX:+UseG1GC -XX:+ExplicitGCInvokesConcurrent -XX:+ParallelRefProcEnabled -XX:+UseStringDeduplication -XX:+UnlockExperimentalVMOptions"
     ```
   - **Network optimization:** Locate agents close to artifact repositories
   - **Master protection:** Move build execution away from master

7. **Monitoring and analysis:**
   - Use plugins like Performance Plugin to track metrics
   - Analyze build logs to identify bottlenecks
   - Implement continuous monitoring of build times

Implementing these strategies can significantly reduce build times and increase pipeline throughput.

## Question 8: How would you implement GitOps with Jenkins?

**Answer:**
Implementing GitOps with Jenkins involves treating Git repositories as the single source of truth for infrastructure and application deployments:

1. **Core GitOps components:**
   - Infrastructure and application configuration stored in Git
   - Jenkins monitoring Git repositories for changes
   - Automated deployment triggered by Git events
   - Desired state reconciliation

2. **Repository structure:**
   - Application code repository
   - Infrastructure repository (Kubernetes manifests, Terraform, etc.)
   - Deployment configuration repository (environment-specific configs)

3. **Jenkins pipeline implementation:**
   ```groovy
   pipeline {
       agent any
       
       triggers {
           // Monitor Git repository for changes
           pollSCM('H/5 * * * *')
       }
       
       stages {
           stage('Checkout') {
               steps {
                   checkout scm
               }
           }
           
           stage('Validate') {
               steps {
                   // Validate Kubernetes manifests
                   sh 'kubeval --strict ./kubernetes/*.yaml'
                   
                   // Lint Helm charts
                   sh 'helm lint ./charts/my-app'
               }
           }
           
           stage('Generate Manifests') {
               steps {
                   // Generate environment-specific manifests
                   sh 'helm template ./charts/my-app -f values-${ENVIRONMENT}.yaml > generated-manifests.yaml'
               }
           }
           
           stage('Deploy') {
               steps {
                   // Apply changes to cluster
                   withKubeConfig([credentialsId: 'kubeconfig']) {
                       sh 'kubectl apply -f generated-manifests.yaml'
                       sh 'kubectl wait --for=condition=available deployment/my-app --timeout=300s'
                   }
               }
           }
           
           stage('Verify') {
               steps {
                   // Verify deployment matches desired state
                   sh './verify-deployment.sh'
                   
                   // Update deployment status in Git
                   withCredentials([usernamePassword(credentialsId: 'git-creds', 
                                   passwordVariable: 'GIT_PASSWORD', 
                                   usernameVariable: 'GIT_USERNAME')]) {
                       sh """
                           git config user.name "Jenkins"
                           git config user.email "jenkins@example.com"
                           echo "${BUILD_NUMBER}" > .deployed-version
                           git add .deployed-version
                           git commit -m "Update deployed version to ${BUILD_NUMBER}"
                           git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/org/repo.git HEAD:${BRANCH_NAME}
                       """
                   }
               }
           }
       }
       
       post {
           failure {
               // Notify on drift detection
               slackSend channel: '#deployments', 
                          color: 'danger', 
                          message: "Deployment drift detected in ${ENVIRONMENT}!"
           }
       }
   }
   ```

4. **Drift detection and reconciliation:**
   - Scheduled jobs to compare actual state with desired state
   - Automatic or manual reconciliation when drift is detected
   ```groovy
   pipeline {
       agent any
       
       triggers {
           // Periodically check for drift
           cron('0 */4 * * *')
       }
       
       stages {
           stage('Detect Drift') {
               steps {
                   checkout scm
                   
                   script {
                       // Compare desired vs actual state
                       def driftDetected = sh(
                           script: './detect-drift.sh',
                           returnStatus: true
                       ) == 1
                       
                       if (driftDetected) {
                           // Option 1: Auto-reconcile
                           sh './reconcile.sh'
                           
                           // Option 2: Manual approval
                           input message: 'Drift detected. Reconcile?'
                           sh './reconcile.sh'
                       }
                   }
               }
           }
       }
   }
   ```

5. **Secret management:**
   - Integration with secrets management tools (Vault, Sealed Secrets)
   - Encrypted secrets stored in Git

6. **Audit and compliance:**
   - All changes tracked in Git history
   - Approval workflows for critical environments
   - Comprehensive audit trail via Git commits

7. **Benefits of this approach:**
   - Versioned and auditable infrastructure changes
   - Self-documenting deployments through Git history
   - Simplified rollbacks to previous states
   - Consistent environments through code reuse

This implementation creates a GitOps workflow where all changes to infrastructure flow through Git repositories and automated pipelines.

## Question 9: Explain how you would implement a dynamic Jenkins agent provisioning strategy across multiple cloud providers.

**Answer:**
Implementing dynamic Jenkins agent provisioning across multiple cloud providers requires:

1. **Multi-cloud plugin configuration:**
   ```groovy
   // Jenkins Configuration as Code (JCasC) example
   jenkins:
     clouds:
       - amazon:
           name: "aws-agents"
           region: "us-west-2"
           sshKeysCredentialsId: "aws-ssh-key"
           templates:
             - description: "AWS Build Agent"
               labelString: "aws linux docker"
               type: "T3Medium"
               remoteFS: "/home/ec2-user"
               mode: EXCLUSIVE
               numExecutors: 1
               idleTerminationMinutes: 10
               
       - kubernetes:
           name: "kubernetes-agents"
           serverUrl: "https://kubernetes.example.com"
           credentialsId: "k8s-credentials"
           templates:
             - name: "k8s-agent"
               label: "k8s java maven"
               containers:
                 - name: "maven"
                   image: "maven:3.8.4-openjdk-11"
                   resourceRequestCpu: "500m"
                   resourceLimitCpu: "1000m"
                   resourceRequestMemory: "2Gi"
                   resourceLimitMemory: "4Gi"
                   
       - azureVM:
           name: "azure-agents"
           azureCredentialsId: "azure-credentials"
           resourceGroup: "jenkins-agents"
           templates:
             - name: "azure-agent"
               labels: "azure windows dotnet"
               location: "eastus"
               virtualMachineSize: "Standard_D2s_v3"
               retentionStrategy:
                 idle:
                   idleTerminationMinutes: 15
   ```

2. **Pipeline integration with dynamic agent selection:**
   ```groovy
   pipeline {
       agent none
       
       stages {
           stage('Build - Java') {
               agent {
                   label 'k8s java maven'
               }
               steps {
                   sh 'mvn clean package'
               }
           }
           
           stage('Build - .NET') {
               agent {
                   label 'azure windows dotnet'
               }
               steps {
                   bat 'dotnet build'
               }
           }
           
           stage('Docker Build') {
               agent {
                   label 'aws linux docker'
               }
               steps {
                   sh 'docker build -t myapp:latest .'
               }
           }
       }
   }
   ```

3. **Cost optimization strategy:**
   - **Spot/Preemptible instances:** Use cheaper, interruptible instances for non-critical builds
   ```groovy
   amazonEC2:
     useSpotInstances: true
     spotMaxBidPrice: '0.15'
   ```
   - **Auto-scaling:** Scale agent pools based on build queue depth
   - **Idle timeout:** Terminate unused agents promptly
   - **Cloud-specific optimizations:** Use reserved instances or savings plans

4. **Agent selection strategy:**
   - **Feature-based selection:** Choose providers based on workload requirements
   ```groovy
   def chooseCloudProvider(Map requirements) {
       if (requirements.containsKey('gpu')) {
           return 'azure-gpu-agents'
       } else if (requirements.containsKey('windows')) {
           return 'aws-windows-agents' 
       } else {
           return 'gcp-linux-agents'
       }
   }
   ```
   - **Cost-based selection:** Select cheapest provider meeting requirements
   - **Geographic optimization:** Choose agents closest to data sources
   - **Fallback mechanism:** Try alternative provider if primary unavailable

5. **Hybrid approach:**
   - **Static core agents:** Maintain minimum pool of permanent agents
   - **Dynamic overflow agents:** Provision cloud agents for peak demand
   - **Specialized agents:** Reserve certain workloads for specific providers

6. **Management and monitoring:**
   ```groovy
   // Monitor cloud agent utilization
   def getCloudAgentStats() {
       def clouds = Jenkins.instance.clouds
       def stats = [:]
       
       clouds.each { cloud ->
           def agentCount = cloud.provisions.size()
           def totalExecutors = cloud.provisions.sum { it.numExecutors ?: 0 }
           
           stats[cloud.name] = [
               'agentCount': agentCount,
               'totalExecutors': totalExecutors,
               'queuedItems': Jenkins.instance.queue.items.findAll { 
                   it.assignedLabel.matches(cloud.labels)
               }.size()
           ]
       }
       
       return stats
   }
   ```
   - Create dashboards for agent utilization across providers
   - Set up alerts for provisioning failures
   - Track costs by provider and build type

7. **Security considerations:**
   - Isolated agent networks per cloud provider
   - Standardized security groups and firewall rules
   - Just-in-time credential provisioning
   - Automated security scanning of agent templates

This comprehensive strategy enables efficient use of multiple cloud providers while optimizing for cost, performance, and availability.

## Question 10: How would you implement canary deployments with progressive traffic shifting using Jenkins?

**Answer:**
Implementing canary deployments with progressive traffic shifting using Jenkins involves:

1. **Infrastructure requirements:**
   - Service mesh (Istio, Linkerd, etc.) or equivalent traffic control mechanism
   - Kubernetes or similar container orchestration
   - Robust monitoring and metrics collection

2. **Jenkins pipeline implementation:**
   ```groovy
   pipeline {
       agent any
       
       environment {
           APP_NAME = 'my-application'
           CANARY_WEIGHT = 0
           MAX_WEIGHT = 100
           WEIGHT_INCREMENT = 20
           ANALYSIS_DURATION = '5m'
       }
       
       stages {
           stage('Deploy Canary') {
               steps {
                   // Deploy canary version alongside stable
                   sh """
                       kubectl apply -f kubernetes/canary/${APP_NAME}-deployment.yaml
                       kubectl rollout status deployment/${APP_NAME}-canary
                   """
               }
           }
           
           stage('Initial Traffic Shift') {
               steps {
                   script {
                       // Initially route small percentage of traffic to canary
                       CANARY_WEIGHT = WEIGHT_INCREMENT
                       applyTrafficWeights(CANARY_WEIGHT)
                   }
                   
                   // Wait for initial metrics
                   sleep time: 60, unit: 'SECONDS'
               }
           }
           
           stage('Progressive Canary') {
               steps {
                   script {
                       // Progressively increase traffic to canary
                       while (CANARY_WEIGHT < MAX_WEIGHT) {
                           // Analyze metrics before increasing traffic
                           def canaryFailed = analyzeMetrics()
                           
                           if (canaryFailed) {
                               echo "Canary analysis failed! Reverting deployment."
                               revertCanary()
                               error "Canary analysis failed at ${CANARY_WEIGHT}% traffic weight."
                           }
                           
                           // Increase traffic percentage
                           CANARY_WEIGHT += WEIGHT_INCREMENT
                           if (CANARY_WEIGHT > MAX_WEIGHT) {
                               CANARY_WEIGHT = MAX_WEIGHT
                           }
                           
                           echo "Increasing canary traffic to ${CANARY_WEIGHT}%"
                           applyTrafficWeights(CANARY_WEIGHT)
                           
                           // Wait for metrics to stabilize
                           sleep time: 3, unit: 'MINUTES'
                       }
                   }
               }
           }
           
           stage('Final Validation') {
               steps {
                   // Extended analysis at 100% traffic
                   script {
                       def canaryFailed = analyzeMetrics(duration: '10m', stricter: true)
                       if (canaryFailed) {
                           revertCanary()
                           error "Final canary analysis failed at 100% traffic."
                       }
                   }
               }
           }
           
           stage('Promote Canary') {
               steps {
                   // Promote canary to stable
                   sh """
                       kubectl apply -f kubernetes/stable/${APP_NAME}-deployment.yaml
                       kubectl delete -f kubernetes/canary/${APP_NAME}-deployment.yaml
                       # Reset to unified traffic routing
                       kubectl apply -f kubernetes/stable/${APP_NAME}-service.yaml
                   """
               }
           }
       }
       
       post {
           failure {
               // Revert on any pipeline failure
               revertCanary()
           }
       }
   }
   
   // Apply traffic weight between stable and canary
   def applyTrafficWeights(int canaryWeight) {
       def stableWeight = 100 - canaryWeight
       
       // Using Istio for traffic control
       sh """
           kubectl apply -f - <<EOF
           apiVersion: networking.istio.io/v1alpha3
           kind: VirtualService
           metadata:
             name: ${APP_NAME}
           spec:
             hosts:
             - ${APP_NAME}.example.com
             http:
             - route:
               - destination:
                   host: ${APP_NAME}-stable
                 weight: ${stableWeight}
               - destination:
                   host: ${APP_NAME}-canary
                 weight: ${canaryWeight}
           EOF
       """
   }
   
   // Analyze metrics to determine canary health
   def analyzeMetrics(Map config = [:]) {
       def duration = config.duration ?: ANALYSIS_DURATION
       def stricter = config.stricter ?: false
       
       // Using Prometheus metrics for analysis
       def errorRateThreshold = stricter ? 0.1 : 0.5
       def latencyThreshold = stricter ? 300 : 500
       
       // Compare error rates between stable and canary
       def stableErrorRate = sh(
           script: """
               curl -s "http://prometheus:9090/api/v1/query" \
               -d 'query=sum(rate(http_requests_total{app="${APP_NAME}-stable",status=~"5.."}[5m]))/sum(rate(http_requests_total{app="${APP_NAME}-stable"}[5m]))' | \
               jq '.data.result[0].value[1]'
           """,
           returnStdout: true
       ).trim().toDouble()
       
       def canaryErrorRate = sh(
           script: """
               curl -s "http://prometheus:9090/api/v1/query" \
               -d 'query=sum(rate(http_requests_total{app="${APP_NAME}-canary",status=~"5.."}[5m]))/sum(rate(http_requests_total{app="${APP_NAME}-canary"}[5m]))' | \
               jq '.data.result[0].value[1]'
           """,
           returnStdout: true
       ).trim().toDouble()
       
       // Compare latency between stable and canary
       def canaryLatency = sh(
           script: """
               curl -s "http://prometheus:9090/api/v1/query" \
               -d 'query=histogram_quantile(0.95, sum(rate(http_request_duration_milliseconds_bucket{app="${APP_NAME}-canary"}[5m])) by (le))' | \
               jq '.data.result[0].value[1]'
           """,
           returnStdout: true
       ).trim().toDouble()
       
       def stableLatency = sh(
           script: """
               curl -s "http://prometheus:9090/api/v1/query" \
               -d 'query=histogram_quantile(0.95, sum(rate(http_request_duration_milliseconds_bucket{app="${APP_NAME}-stable"}[5m])) by (le))' | \
               jq '.data.result[0].value[1]'
           """,
           returnStdout: true
       ).