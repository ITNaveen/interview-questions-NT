# C# CI/CD Pipeline with SonarQube, Trivy, and ArgoCD

## 1. Overview of My CI/CD Pipeline

As a DevOps Engineer, I've designed and implemented a robust CI/CD pipeline for our C# applications that integrates code quality analysis, security scanning, containerization, and GitOps-based deployment. This pipeline consists of the following stages:

```
GitHub -> Jenkins -> Build -> SonarQube -> Artifactory -> Docker Build -> Trivy -> Docker Push -> K8s Repo Update -> ArgoCD -> Staging -> Production
```

This pipeline ensures:
- Code quality validation through SonarQube
- Artifact management in Artifactory
- Container security scanning with Trivy
- Automated deployment using GitOps principles with ArgoCD
- Controlled promotion from staging to production

## 2. Source Control and Branching Strategy

### 2.1 GitHub Repository Structure
We maintain two separate repositories:
- **Application Repository**: Contains C# application code, Dockerfile, Jenkins pipeline definition, and SonarQube configuration
- **Kubernetes Repository**: Contains Kubernetes manifests (deployments, services, etc.) for our application environments

### 2.2 Branching Strategy
We follow a trunk-based development approach with:
In Trunk-Based Development (TBD), you:
1️⃣ Create a short-lived feature branch from main.
2️⃣ Work on it for 1-3 days (max).
3️⃣ Merge it back to main quickly.
4️⃣ Delete the feature branch after merging.

## 3. Jenkins Pipeline Configuration

### 3.1 Jenkinsfile for C# Application
```yml
pipeline {
    agent any
    
    environment {
        DOTNET_CLI_HOME = '/tmp/dotnet_cli_home'
        DOCKER_REGISTRY = 'registry.example.com'
        APP_NAME = 'csharp-app'
        SONAR_PROJECT_KEY = 'company:csharp-app'
        KUBERNETES_REPO = 'git@github.com:company/kubernetes-manifests.git'
        ARTIFACTORY_URL = 'https://artifactory.example.com/artifactory'
        ARTIFACTORY_REPO = 'csharp-apps'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Restore & Build') {
            steps {
                sh 'dotnet restore'  # This downloads and installs any missing NuGet packages listed in the .csproj
                sh 'dotnet build --no-restore --configuration Release' # This ensures that all source files are compiled, and an output (DLLs or EXEs) is generated.
            }        # Compilation is the process of converting human-readable C# source code (.cs files) into machine-readable binary code (DLLs or EXEs).
        }
        
        stage('Unit Tests') {
            steps {
                sh 'dotnet test --no-build --configuration Release --collect:"XPlat Code Coverage"'
            }
            post {
                always {
                    step([$class: 'CoberturaPublisher', coberturaReportFile: '**/coverage.cobertura.xml'])
                }
            }
        }
What Do Unit Tests Check?
Unit tests verify small, isolated parts of the code to ensure they behave as expected.
Here’s what unit tests typically check:
✅ Business logic correctness (e.g., does a method return the expected result?)
✅ Edge cases and error handling (e.g., does it handle null values properly?)
✅ Mathematical or logical operations (e.g., does a calculation return the correct value?)
✅ Functionality of classes/methods in isolation (without database, API calls, etc.)
unit test is placed under test dir as .cs extention.

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        dotnet sonarscanner begin \
                          /k:"${SONAR_PROJECT_KEY}" \    // Sets the unique SonarQube project key
                          /d:sonar.cs.opencover.reportsPaths="**/coverage.opencover.xml" \      //Loads code coverage data from unit tests.
                          /d:sonar.cs.vstest.reportsPaths="**/TestResults/*.trx" \              // Loads unit test results from the trx test report.
                          /d:sonar.coverage.exclusions="**Tests*.cs,**/Program.cs" \            //Excludes test files and Program.cs from coverage analysis.
                          /d:sonar.cpd.exclusions="**/Migrations/*.cs" \                        //Excludes duplicate code detection for Entity Framework Migrations.
                          /d:sonar.cs.roslyn.reportPaths="**/analyzer-results.json" \           //Integrates Roslyn Analyzers (Microsoft's static code analysis tool).
                          /d:sonar.exclusions="**/obj/**,**/bin/**"         //Excludes compiled files (bin/, obj/) from analysis.
                        
                        dotnet build --no-restore --configuration Release
                        
                        dotnet sonarscanner end   //Finalizes the SonarQube scan and sends results to the SonarQube server.
                    '''
                }
            }
        }
How It Works in Jenkins

    Jenkins triggers the SonarScanner during a build.
    SonarScanner analyzes the code and pushes the results to SonarQube.
    You check the results in the SonarQube UI.

SonarQube checks for:
1. Bugs:
SonarQube looks for logical errors or issues in the code that could cause problems during runtime, like null pointer exceptions (SonarQube scans the code without executing it. It doesn't need to run the program to identify potential issues. It looks for code patterns that could lead to errors, like accessing an object that could be null.) or incorrect calculations.
2. Security Vulnerabilities:
It checks for security flaws, such as risks for SQL injection, cross-site scripting (XSS), or storing sensitive data like passwords in the code.
3. Code Smells:
SonarQube identifies "bad smells" in the code — things that aren’t necessarily errors but could make the code hard to understand or maintain in the future, like long methods or duplicated code.
4. Code Duplication:
It flags sections of the code that are duplicated, encouraging developers to refactor and reduce repetition, making the code more efficient and easier to maintain.
5. Test Coverage:
SonarQube checks if the code is properly tested by unit tests. It reports the percentage of the code covered by tests to ensure that important parts are well-tested and bugs are less likely.


Key Difference Between Unit Tests & SonarQube
🔍 Unit Tests (Functional Testing)	                                   🛠 SonarQube (Code Quality Analysis)
✅ Checks code functionality (Does the code work as expected?)	      ✅ Checks code quality, security, and maintainability
✅ Runs actual test cases against functions/classes	                  ✅ Static code analysis (without running the program)
✅ Uses test frameworks like xUnit, NUnit, MSTest	                  ✅ Uses SonarQube rules and analysis tools
✅ Detects logic errors (e.g., incorrect calculations)	              ✅ Detects bad coding practices (e.g., duplicate code, security flaws)
✅ Generates test reports (Pass/Fail results)	                      ✅ Generates code quality reports (bugs, vulnerabilities, code smells)
❌ Does not check for security issues or code duplication	          ❌ Does not validate whether functions work correctly
        
        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package and Publish to Artifactory') {
            steps {
                sh 'dotnet publish -c Release -o publish'
                
                rtUpload (
                    serverId: 'artifactory',
                    spec: '''{
                        "files": [
                            {
                                "pattern": "publish/**",
                                "target": "${ARTIFACTORY_REPO}/${APP_NAME}/${env.BUILD_NUMBER}/"
                            }
                        ]
                    }''',
                    buildName: "${APP_NAME}",
                    buildNumber: "${env.BUILD_NUMBER}"
                )
                
                rtPublishBuildInfo (
                    serverId: 'artifactory',
                    buildName: "${APP_NAME}",
                    buildNumber: "${env.BUILD_NUMBER}"
                )
            }
        }
        
        stage('Docker Build') {
            steps {
                rtDownload (
                    serverId: 'artifactory',
                    spec: '''{
                        "files": [
                            {
                                "pattern": "${ARTIFACTORY_REPO}/${APP_NAME}/${env.BUILD_NUMBER}/**",
                                "target": "app/"
                            }
                        ]
                    }'''
                )
                
                sh """
                    docker build -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT.substring(0,8)} \
                      --build-arg VERSION=${env.BUILD_NUMBER} .
                """
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh """
                    trivy image \
                      --format json \
                      --output trivy-results.json \
                      --severity HIGH,CRITICAL \
                      ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT.substring(0,8)}
                """
                
                script {
                    def trivyResult = readJSON file: 'trivy-results.json'
                    def vulnerabilityCount = 0
                    
                    trivyResult.Results.each { result ->
                        if (result.Vulnerabilities) {
                            vulnerabilityCount += result.Vulnerabilities.size()
                        }
                    }
                    
                    if (vulnerabilityCount > 0) {
                        error "Found ${vulnerabilityCount} HIGH or CRITICAL vulnerabilities in the container"
                    }
                }
Trivy scans Docker images by checking OS packages and application dependencies against databases like GitHub Security Advisories, NVD, and OS security trackers. For C# projects, it looks at NuGet package vulnerabilities and CVEs in dependencies. If Trivy finds HIGH or CRITICAL vulnerabilities, we fail the pipeline to ensure secure deployments.
🛠️ Trivy is a security tool that checks for problems in your .NET app.
✅ It looks at system software inside your app (Windows/Linux packages).
✅ It scans NuGet packages (external .NET libraries your app uses).
✅ It finds security issues (known vulnerabilities hackers can exploit).

                // Publish scan results to Artifactory
                rtUpload (
                    serverId: 'artifactory',
                    spec: '''{
                        "files": [
                            {
                                "pattern": "trivy-results.json",
                                "target": "${ARTIFACTORY_REPO}/${APP_NAME}/${env.BUILD_NUMBER}/security-scans/"
                            }
                        ]
                    }'''
                )
            }
        }
        
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-registry', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login ${DOCKER_REGISTRY} -u ${DOCKER_USERNAME} --password-stdin
                        docker push ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT.substring(0,8)}
                    '''
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        mkdir -p ~/.ssh
                        ssh-keyscan github.com >> ~/.ssh/known_hosts
                        GIT_SSH_COMMAND="ssh -i $SSH_KEY" git clone ${KUBERNETES_REPO} k8s-repo
                        cd k8s-repo/staging
                        
                        # Update image tag in deployment.yaml
                        sed -i "s|image: ${DOCKER_REGISTRY}/${APP_NAME}:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT.substring(0,8)}|g" deployment.yaml
                        
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git add .
                        git commit -m "Update ${APP_NAME} image to ${env.GIT_COMMIT.substring(0,8)}"
                        GIT_SSH_COMMAND="ssh -i $SSH_KEY" git push origin main
                    '''
                }
            }
        }
        
        stage('Wait for ArgoCD Sync') {
            steps {
                sh '''
                    # Poll ArgoCD for sync status
                    for i in $(seq 1 30); do
                        sync_status=$(argocd app get ${APP_NAME}-staging -o json | jq -r '.status.sync.status')
                        health_status=$(argocd app get ${APP_NAME}-staging -o json | jq -r '.status.health.status')
                        
                        if [ "$sync_status" = "Synced" ] && [ "$health_status" = "Healthy" ]; then
                            echo "ArgoCD sync and health check successful"
                            break
                        fi
                        
                        if [ $i -eq 30 ]; then
                            echo "Timeout waiting for ArgoCD sync"
                            exit 1
                        fi
                        
                        echo "Waiting for ArgoCD sync to complete... ($sync_status / $health_status)"
                        sleep 10
                    done
                '''
            }
        }

The approval process happens in Jenkins, where the pipeline pauses using the input step. A user must manually approve the deployment in Jenkins UI. Once approved, the pipeline updates the production manifest in Git, and ArgoCD automatically syncs the change to deploy it

        stage('Promote to Production') {
            steps {
                input message: 'Promote to production?', ok: 'Approve'
                
                withCredentials([sshUserPrivateKey(credentialsId: 'github-ssh', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        cd k8s-repo/production
                        
                        # Update image tag in production deployment
                        sed -i "s|image: ${DOCKER_REGISTRY}/${APP_NAME}:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.GIT_COMMIT.substring(0,8)}|g" deployment.yaml
                        
                        git config user.email "jenkins@example.com"
                        git config user.name "Jenkins"
                        git add .
                        git commit -m "Promote ${APP_NAME} image ${env.GIT_COMMIT.substring(0,8)} to production"
                        GIT_SSH_COMMAND="ssh -i $SSH_KEY" git push origin main
                    '''
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```


## 4. SonarQube for C# Applications: In-Depth Analysis

### 4.1 Test Types and Analysis in SonarQube for C#

SonarQube performs several types of analysis on C# code:

1. **Static Code Analysis**: Examines code without executing it
   - **Syntax Analysis**: Checks for syntax errors and coding standard violations
   - **Semantic Analysis**: Identifies logical errors and anti-patterns
   - **Control Flow Analysis**: Detects unreachable code and logic issues
   - **Data Flow Analysis**: Tracks variable values to find bugs

2. **Test Coverage Analysis**: Measures code covered by tests
   - **Line Coverage**: Percentage of code lines executed during tests
   - **Branch Coverage**: Percentage of branches (if/else paths) executed
   - **Method Coverage**: Percentage of methods called during tests

3. **Code Duplication Analysis**: Finds repeated code blocks

4. **Code Security Analysis**: Identifies security vulnerabilities
   - **OWASP Top 10**: SQL Injection, XSS, CSRF, etc.
   - **CWE**: Common Weakness Enumeration checks
   - **.NET-specific**: Security issues related to .NET framework

5. **Code Complexity Analysis**: Measures code maintainability
   - **Cyclomatic Complexity**: Number of linearly independent paths
   - **Cognitive Complexity**: Measures how difficult code is to understand
   - **Class Complexity**: Combined complexity of methods in a class

### 4.2. C# Specific Rules and Analyzers

SonarQube uses multiple analyzers for C# code:

1. **SonarC# Analyzer**: SonarSource's own analyzer with 400+ rules covering:
   - Code smells (unused variables, redundant casts)
   - Bugs (null reference exceptions, resource leaks)
   - Vulnerabilities (SQL injection, hardcoded credentials)
   - Maintainability issues (complex methods, commented code)

2. **Roslyn Analyzers**: Microsoft's .NET Compiler Platform analyzers
   - Language usage rules
   - API design guidelines
   - Performance best practices
   - .NET Framework usage rules

3. **StyleCop Analyzers**: Style and consistency rules
   - Documentation requirements
   - Spacing and formatting rules
   - Naming conventions
   - Code organization

4. **FxCop Analyzers**: Framework design and usage rules
   - Security rules
   - Performance rules
   - Maintainability rules
   - Reliability rules

### 4.3 SonarQube Configuration for C# Projects

My specific SonarQube configuration involves:

#### 4.3.1 SonarQube Scanner Setup

We use the .NET SonarScanner, which integrates with MSBuild:

```bash
dotnet tool install --global dotnet-sonarscanner
```

#### 4.3.2 Key Configuration Parameters

```
/k:"${SONAR_PROJECT_KEY}"                        # Project key for identification
/d:sonar.cs.opencover.reportsPaths="..."         # Test coverage report paths
/d:sonar.cs.vstest.reportsPaths="..."            # Test execution report paths
/d:sonar.coverage.exclusions="..."               # Files to exclude from coverage
/d:sonar.cpd.exclusions="..."                    # Files to exclude from duplication
/d:sonar.cs.roslyn.reportPaths="..."             # Roslyn analyzer results
/d:sonar.exclusions="..."                        # Files to exclude from analysis
```

#### 4.3.3 Custom Quality Gate Configuration

Our customized Quality Gate includes:

1. **Reliability (Bugs):**
   - No new critical or blocker bugs
   - Bug density < 0.05 per 1000 lines of code

2. **Security:**
   - No new critical or blocker vulnerabilities
   - No security hotspots without review

3. **Maintainability:**
   - Technical debt ratio < 5%
   - Maintainability rating ≥ B
   - Code smells < 0.1 per 1000 lines of code

4. **Coverage:**
   - Overall coverage ≥ 80%
   - New code coverage ≥ 85%
   - No uncovered conditions on new code

5. **Duplications:**
   - Duplication < 3% of lines
   - No duplicated blocks

### 4.4 Daily Usage and Monitoring

1. **Pre-commit Analysis:**
   - Developers use SonarLint IDE integration
   - Automated analysis in pull requests

2. **Quality Gate Monitoring:**
   - Jenkins waits for Quality Gate results
   - Failed gates block merge to main branch

3. **Technical Debt Management:**
   - Weekly debt reduction sessions
   - Prioritization based on severity and impact

4. **Custom Dashboards:**
   - Team-specific quality metrics
   - Trend analysis for long-term improvements

## 5. Artifactory Integration

### 5.1 Build Artifact Management

Our pipeline publishes .NET build artifacts to Artifactory:

1. **Publishing Process:**
   - `dotnet publish` creates release binaries
   - Jenkins uploads binaries to Artifactory repository
   - Build info is attached for traceability

2. **Artifact Structure:**
   ```
   artifactory/csharp-apps/
   └── app-name/
       └── build-number/
           ├── bin/
           │   └── (application binaries)
           ├── configs/
           │   └── (configuration files)
           └── libs/
               └── (dependency assemblies)
   ```

3. **Artifact Metadata:**
   - Build number and commit hash
   - Environment variables
   - Build duration and timestamp
   - Dependencies and modules

### 5.2 Artifact Promotion Strategy

Artifacts progress through promotion paths:

1. **Initial Repository:** `csharp-apps-dev`
2. **After Successful Testing:** `csharp-apps-staging`
3. **Production Release:** `csharp-apps-release`

## 6. Docker Build with Artifacts from Artifactory

### 6.1 Dockerfile Configuration

Our Dockerfile is designed to work with pre-built artifacts from Artifactory:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Copy pre-built application from Artifactory
COPY app/ .

# Configure environment
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_EnableDiagnostics=0

# Set non-root user for security
RUN adduser --disabled-password --gecos "" appuser && \
    chown -R appuser:appuser /app
USER appuser

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 6.2 Build Process

The Docker build process:

1. Downloads artifacts from Artifactory to local workspace
2. Builds image with appropriate tags (commit hash, build number)
3. Configures environment settings
4. Implements security best practices (non-root user, minimal base image)

## 7. Trivy Security Scanning

### 7.1 Types of Scans Performed by Trivy

Trivy performs comprehensive security scans on our Docker images:

1. **OS Package Vulnerability Scanning:**
   - Debian/Ubuntu packages in base images
   - Alpine apk packages
   - RedHat/CentOS RPM packages

2. **Application Dependency Scanning:**
   - .NET NuGet packages
   - npm packages (for any JavaScript components)
   - Language-specific libraries

3. **Misconfiguration Scanning:**
   - Dockerfile best practices
   - Kubernetes manifest security
   - Infrastructure-as-Code configurations

4. **Secret Scanning:**
   - Detects hardcoded API keys
   - Identifies embedded credentials
   - Finds private keys and certificates

5. **License Scanning:**
   - Identifies open-source licenses
   - Flags non-compliant or risky licenses

### 7.2 Trivy Configuration for C# Applications

Our specific Trivy configuration includes:

#### 7.2.1 Command Line Options

```bash
trivy image \
  --format json \                        # Output format
  --output trivy-results.json \          # Output file
  --severity HIGH,CRITICAL \             # Severity filtering
  --vuln-type os,library \               # Vulnerability types
  --ignore-unfixed \                     # Skip vulnerabilities without fixes
  --timeout 10m \                        # Scan timeout
  --cache-dir /var/lib/trivy \           # Cache location
  ${DOCKER_REGISTRY}/${APP_NAME}:${TAG}  # Target image
```

#### 7.2.2 Custom Policies and Exclusions

We maintain a `.trivyignore` file for our C# application images:

```
# Vulnerabilities pending patches in .NET dependencies
CVE-2021-XXXXX
CVE-2022-XXXXX

# False positives for our implementation
CVE-2023-XXXXX
```

#### 7.2.3 Vulnerability Management Process

1. **Severity Handling:**
   - CRITICAL/HIGH: Block deployment
   - MEDIUM: Review within 2 weeks
   - LOW: Document in backlog

2. **Remediation Workflow:**
   - Emergency patch for exploitable vulnerabilities
   - Scheduled updates for vulnerable dependencies
   - Compensating controls documentation

### 7.3 Integration with Security Processes

1. **Security Gate:**
   - Failed scans block deployment
   - Results archived in Artifactory
   - Security team notified of findings

2. **Compliance Reporting:**
   - Scan results mapped to compliance frameworks
   - Evidence collection for audits
   - Vulnerability aging reports

3. **Continuous Monitoring:**
   - Daily rescans of production images
   - Monitoring for new CVEs affecting deployed images
   - Automatic remediation tickets

## 8. Docker Registry Integration

After successful Trivy scanning, our images are:

1. Tagged with both git commit hash and semantic version
2. Pushed to our private Docker registry
3. Metadata attached (build info, scan results)
4. Signed with Docker Content Trust

## 9. GitOps with ArgoCD

### 9.1 Kubernetes Repository Structure

```
kubernetes-repo/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── configmap.yaml
├── staging/
│   ├── kustomization.yaml
│   └── environment-config.yaml
└── production/
    ├── kustomization.yaml
    └── environment-config.yaml
```

### 9.2 ArgoCD Configuration

Our ArgoCD applications are defined as:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: csharp-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'git@github.com:company/kubernetes-manifests.git'
    path: staging
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 9.3 Promotion Workflow

1. **Staging Deployment:**
   - Automatic synchronization by ArgoCD
   - Health checks confirm successful deployment
   - Integration tests run against staging environment

2. **Production Approval:**
   - Manual approval in Jenkins
   - Image tag updated in production manifests
   - Change committed to Kubernetes repository

3. **Production Deployment:**
   - ArgoCD detects change and syncs
   - Progressive rollout with monitoring
   - Automated rollback on health check failure

## 10. Real-World Experiences and Interview Q&A

### 10.1 Common Issues and Resolutions

#### SonarQube Challenges:

1. **Issue**: False positives in generated code
   **Solution**: Configured exclusion patterns for auto-generated files

2. **Issue**: Quality gate timeouts in large codebases
   **Solution**: Implemented incremental analysis and optimized database

3. **Issue**: Integration with .NET 6 minimal APIs
   **Solution**: Custom rule adjustments for new architecture patterns

#### Trivy Challenges:

1. **Issue**: High number of unfixed vulnerabilities
   **Solution**: Implemented risk-based assessment and compensating controls

2. **Issue**: Slow scans in CI pipeline
   **Solution**: Optimized cache configuration and parallel scanning

3. **Issue**: False positives in complex multi-stage builds
   **Solution**: Created custom policies and targeted scanning

### 10.2 Interview Q&A

#### SonarQube Questions:

**Q: How do you handle SonarQube quality gates for legacy code?**  
A: For legacy C# applications, I've implemented a phased approach. First, I establish a baseline by analyzing the current state and setting initial thresholds that match current quality. Then I create stricter quality gates that only apply to new code, ensuring we don't introduce new issues while gradually improving the legacy codebase. I've also configured exclusion patterns for generated code (like Entity Framework migrations) and implemented technical debt reduction sprints focused on high-impact areas.

**Q: What SonarQube metrics do you find most valuable for C# applications?**  
A: For our C# applications, I've found that cognitive complexity, code coverage, and security hotspots provide the most actionable insights. Cognitive complexity helps identify methods that are difficult to maintain, which is particularly important in C# where LINQ and async/await can hide complexity. Code coverage with branch coverage is essential for validating our test effectiveness, especially in business logic layers. Security hotspots have been valuable for identifying potential issues in our API controllers and data access layers, particularly around SQL injection and CSRF vulnerabilities.

**Q: How do you configure SonarQube for microservices architecture?**  
A: In our microservices environment, I've set up a multi-project configuration in SonarQube with a hierarchical structure. Each C# microservice has its own project with service-specific quality profiles, while sharing a common core of rules. I've implemented a custom dashboard that shows the health of the entire system while allowing drill-down into individual services. For shared libraries, we have stricter quality gates since issues there impact multiple services. I've also configured the portfolio feature to track overall quality metrics across all services.

#### Trivy Questions:

**Q: How do you handle Trivy findings in base images you can't modify?**  
A: This is a common challenge with .NET base images. I've implemented a risk-based approach where I first assess the exploitability of the vulnerability in our context. For vulnerabilities in base images, I document the risk assessment with compensating controls like network policies and runtime protection. We maintain a vulnerability exception process that requires security team approval and regular reviews. For critical vulnerabilities, we sometimes create custom base images with security patches applied or implement application-level workarounds.

**Q: How do you integrate Trivy results into your overall security posture?**  
A: We've implemented a comprehensive approach where Trivy results are part of our security dashboard alongside SonarQube security findings, DAST results, and infrastructure scans. I've set up integration between Trivy and our ticket system to automatically create remediation tasks with appropriate severity. We use Artifactory's XRay feature alongside Trivy for additional validation, and I've configured our monitoring system to alert on new critical CVEs that affect our production images, enabling rapid response.

**Q: How do you optimize Trivy scans in CI/CD for large applications?**  
A: To optimize Trivy scanning in our CI/CD pipeline, I've implemented several techniques. First, we use a centralized cache server to avoid redundant downloads of vulnerability databases. Second, I've configured targeted scanning for specific layers to focus on our application code rather than repeatedly scanning unchanged base layers. Third, we run parallel scans for multiple variants of our images. Finally, I've implemented smart filtering that only blocks the pipeline for exploitable vulnerabilities rather than theoretical ones, while still tracking all issues for future remediation.

#### CI/CD and Deployment Questions:

**Q: How do you handle database migrations in your CI/CD pipeline?**  
A: For our C# applications using Entity Framework, I've implemented a separation between application deployment and database migrations. In the CI pipeline, we generate migration scripts and validate them with tests. Before deployment, we run a separate job that applies migrations using a dedicated service account with appropriate permissions. For production, we generate a migration plan that's reviewed before execution, and we implement transaction rollback capability for failed migrations. We also maintain database snapshots before significant schema changes as an additional safety measure.

**Q: How do you manage secrets across your pipeline?**  
We use a multi-layered approach to secrets management. For CI/CD, Jenkins credentials are securely stored in AWS Secrets Manager and retrieved at runtime using the AWS CLI or Jenkins plugins. Application secrets are also managed in AWS Secrets Manager with fine-grained IAM policies, ensuring least-privilege access. In Kubernetes, we use the External Secrets Operator to fetch secrets dynamically from AWS Secrets Manager and inject them into pods as environment variables or Kubernetes secrets. Our security pipeline includes Trivy scans to detect exposed credentials in images and SonarQube to check for hardcoded secrets in the codebase.

**Q: How do you ensure zero-downtime deployments with your ArgoCD setup?**  
"We ensure zero-downtime deployments using the Blue-Green Deployment strategy. In this approach, we deploy a new version (Blue) while keeping the current version (Green) running. Once the Blue version is tested and verified, we switch all traffic to it and shut down the Green version. This ensures that users always interact with a fully running application, eliminating downtime."

"We also configure liveness and readiness probes to ensure smooth transitions:"

    Readiness Probe 🟢 → Ensures the new pod is fully started before receiving traffic.
    Liveness Probe 🔴 → Detects if a pod becomes unresponsive and automatically restarts it.

"Compared to other deployment strategies, Blue-Green provides a fast rollback mechanism—if an issue is detected after switching, we can instantly revert traffic back to the previous version. This makes it highly reliable for production deployments."
