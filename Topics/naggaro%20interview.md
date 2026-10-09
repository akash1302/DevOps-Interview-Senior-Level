# Naggaro Interview --- Senior DevOps Interview Q&A

## 1. Design a 3-tier architecture. Explain the workflow step by step.

### Answer

For a typical production application, I would keep the architecture in
three layers:

``` text
Users
  |
  v
Route53
  |
  v
CloudFront / WAF
  |
  v
Public ALB
  |
  v
Application Tier
(EKS / ECS / EC2 - Private Subnets)
  |
  v
Database Tier
(RDS/Aurora - Private DB Subnets)
```

I normally deploy it across multiple Availability Zones for high
availability.

### Workflow

1.  The user accesses the application using the domain name.
2.  Route53 resolves the domain to the application endpoint.
3.  If CloudFront is used, the request first reaches CloudFront. WAF
    filters malicious traffic such as common web attacks.
4.  The request reaches the Application Load Balancer in the public
    subnets.
5.  ALB terminates HTTPS and forwards the request to the required
    application service.
6.  The application runs in private subnets. In Kubernetes, this could
    be EKS with pods behind an internal Kubernetes Service.
7.  The application talks to RDS/Aurora in private database subnets.
8.  Security groups allow only the required traffic. For example, ALB
    -\> application on 8080 and application -\> database on 5432.
9.  Database subnets do not have direct internet access.
10. Application logs and metrics are sent to CloudWatch/ELK, and alerts
    are configured for important failures.

For production, I would also add Multi-AZ, autoscaling, backups,
monitoring, centralized logging, IAM least privilege, encryption, and
disaster recovery.

------------------------------------------------------------------------

## 2. How does CI/CD work? What commands do you run during a Maven build?

### Answer

I normally split CI/CD into two parts:

-   **CI:** checkout code, build, test, scan, package and create an
    artifact/container image.
-   **CD:** deploy the approved artifact to Kubernetes and verify the
    deployment.

For a Java Maven application, a typical pipeline is:

``` text
Developer
   |
   v
Git push / Pull Request
   |
   v
CI Pipeline
   |
   +--> Checkout
   +--> Maven build
   +--> Unit tests
   +--> SonarQube / SAST
   +--> Package JAR
   +--> Docker build
   +--> Image scan
   +--> Push image to ECR
   |
   v
CD
   |
   +--> Deploy to Dev
   +--> Smoke test
   +--> Approval
   +--> Deploy to Prod
   +--> Health verification
```

### Maven commands I commonly use

``` bash
mvn clean
mvn test
mvn package
```

In a CI pipeline I would normally use:

``` bash
mvn clean verify
```

If I need to skip tests temporarily for a specific build:

``` bash
mvn clean package -DskipTests
```

For a normal production pipeline, I would not skip tests unless there is
a documented reason.

Other useful commands:

``` bash
mvn dependency:tree
mvn clean install
mvn spring-boot:run
```

The important point is that I don't just run `mvn package` and deploy. I
also validate code quality, security, the generated artifact, and the
container image before deployment.

------------------------------------------------------------------------

## 3. How does GitHub work? How do you clone, pull, and commit code?

### Answer

Git is the version control system, while GitHub is a platform that hosts
Git repositories and provides collaboration features such as pull
requests, reviews, branch protection and CI/CD integration.

### Clone a repository

``` bash
git clone https://github.com/company/app.git
cd app
```

### Check the current state

``` bash
git status
git branch
git remote -v
```

### Create a feature branch

``` bash
git checkout -b feature/login-fix
```

Or:

``` bash
git switch -c feature/login-fix
```

### Pull the latest changes

``` bash
git pull origin main
```

For a feature branch:

``` bash
git pull origin feature/login-fix
```

### Stage and commit

``` bash
git add .
git commit -m "Fix login validation"
```

### Push

``` bash
git push origin feature/login-fix
```

Then I create a Pull Request in GitHub, run CI checks, get the code
reviewed, and merge it according to the branch protection rules.

### Practical point

Before starting work, I usually pull the latest code or rebase my branch
so that I don't work on stale code. For production repositories, I
prefer PR-based changes instead of allowing direct pushes to `main`.

------------------------------------------------------------------------

## 4. What is PDB (Pod Disruption Budget)?

### Answer

A PodDisruptionBudget controls how many pods can be voluntarily
disrupted at the same time.

For example, if I have:

``` yaml
replicas: 3
```

and I define:

``` yaml
minAvailable: 2
```

Kubernetes should keep at least two pods available during voluntary
disruptions such as:

-   Node drain
-   Cluster maintenance
-   Node upgrade
-   Kubernetes upgrade

Example:

``` yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: java-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: java-app
```

PDB does **not** protect against every type of failure. For example, if
a node suddenly crashes, that is an involuntary disruption.

### Practical approach

For a production application, I normally use multiple replicas, a PDB,
readiness probes and proper rolling-update settings together. PDB alone
does not guarantee zero downtime.

------------------------------------------------------------------------

## 5. What are readiness and liveness probes? What do you add in deployment.yaml?

### Answer

I use the probes for two different purposes.

### Readiness probe

Readiness answers:

> "Can this pod receive traffic right now?"

If readiness fails, Kubernetes removes the pod from the Service
endpoints, so new traffic is not sent to it.

This is important during startup, deployments and temporary dependency
problems.

### Liveness probe

Liveness answers:

> "Is the application still alive, or is it stuck?"

If liveness repeatedly fails, Kubernetes restarts the container.

### Example

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app

  template:
    metadata:
      labels:
        app: java-app

    spec:
      containers:
        - name: java-app
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/java-app:1.0.0
          ports:
            - containerPort: 8080

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
            timeoutSeconds: 3
            failureThreshold: 3
```

I normally avoid making the liveness probe depend on external services
such as a database. If the database has a temporary issue, I don't want
Kubernetes restarting every application pod unnecessarily.

------------------------------------------------------------------------

## 6. After building an application in CI/CD, what practical steps do you perform next?

### Answer

After the application build succeeds, I don't directly push it to
production.

My usual flow is:

``` text
Build
  |
  v
Unit Tests
  |
  v
Code Quality / Security Scan
  |
  v
Create JAR
  |
  v
Build Docker Image
  |
  v
Scan Image
  |
  v
Push Image to ECR
  |
  v
Deploy to Dev
  |
  v
Smoke / Health Tests
  |
  v
Approval
  |
  v
Deploy to Prod
  |
  v
Verify + Monitor
```

### Practical steps

1.  Generate the application artifact, for example a JAR.
2.  Build the Docker image.
3.  Tag the image with an immutable version such as the Git commit SHA.
4.  Scan the image for vulnerabilities.
5.  Push the image to ECR.
6.  Update the Kubernetes deployment with the new image.
7.  Deploy to Dev first.
8.  Verify pods, deployment status, application health and logs.
9.  Run smoke/API tests.
10. Promote the same image to production.
11. Monitor error rate, latency, CPU, memory and application logs.
12. If there is a problem, rollback to the previous known-good image.

I prefer promoting the **same tested image** from Dev to Prod rather
than rebuilding the application separately for Prod.

------------------------------------------------------------------------

## 7. Write a deployment.yaml manifest.

### Answer

A basic production-style Deployment could look like this:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
  namespace: dev
  labels:
    app: java-app

spec:
  replicas: 3

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  selector:
    matchLabels:
      app: java-app

  template:
    metadata:
      labels:
        app: java-app

    spec:
      containers:
        - name: java-app
          image: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/java-app:1.0.0

          ports:
            - containerPort: 8080

          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1"
              memory: "1Gi"

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "dev"

          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true

      terminationGracePeriodSeconds: 30
```

For production, I would also consider:

-   PodDisruptionBudget
-   topology spread constraints / pod anti-affinity
-   HPA
-   Secrets from AWS Secrets Manager or another secret-management
    solution
-   ConfigMaps
-   NetworkPolicies
-   ServiceAccount/IAM role
-   proper resource sizing
-   namespace-specific configuration

------------------------------------------------------------------------

## 8. Write a multi-stage Dockerfile.

### Answer

For a Java Maven application, I would use a multi-stage Dockerfile so
the final image contains only what is required to run the application.

``` dockerfile
# Stage 1 - Build
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline

COPY src ./src

RUN mvn clean package -DskipTests


# Stage 2 - Runtime
FROM eclipse-temurin:17-jre

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

RUN useradd -r -u 10001 appuser
USER 10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Why multi-stage?

The Maven compiler and dependencies are required only during the build.

The final runtime image needs only the JRE and application JAR.

This makes the final image smaller and reduces the attack surface.

In a real production pipeline, I would also pin base images
appropriately, scan the image, avoid running as root, and use an
immutable image tag.

------------------------------------------------------------------------

## 9. If a pod crashes, what changes would you make in deployment.yaml?

### Answer

I would not immediately change the Deployment just because a pod
crashed. First I would find the reason.

I normally check:

``` bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
kubectl get events --sort-by=.lastTimestamp
```

`--previous` is particularly useful when the container has already
restarted.

I check whether the problem is:

-   OOMKilled
-   application exception
-   failed liveness probe
-   bad configuration
-   missing Secret/ConfigMap
-   image issue
-   dependency failure
-   CPU/memory pressure
-   node problem

### If it is an OOMKilled issue

I would review and potentially increase memory requests/limits:

``` yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

I would not blindly increase memory. I would first confirm the actual
memory usage and JVM behavior.

### If the application needs more startup time

I may use a `startupProbe`:

``` yaml
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

This is useful for slow-starting Java applications because Kubernetes
gives the application time to start before liveness checking becomes
active.

### If the issue is a bad deployment

I would rollback:

``` bash
kubectl rollout history deployment/java-app
kubectl rollout undo deployment/java-app
kubectl rollout status deployment/java-app
```

The key point is: **I troubleshoot the root cause first and then change
the manifest based on evidence.**

------------------------------------------------------------------------

## 10. What is the difference between ADD and COPY in Docker?

### Answer

Both `COPY` and `ADD` can copy files into an image.

I normally prefer `COPY` because it is simpler and has predictable
behavior.

### COPY

``` dockerfile
COPY app.jar /app/app.jar
```

It simply copies files/directories from the build context into the
image.

### ADD

`ADD` has additional behavior. For example, it can extract local tar
archives and has support for URL sources in Dockerfile semantics.

Example:

``` dockerfile
ADD application.tar.gz /app/
```

### Practical rule

If I only need to copy files, I use:

``` dockerfile
COPY
```

I use `ADD` only when I specifically need its extra functionality.

For most production Dockerfiles, `COPY` is the better default because
the intent is clear.

------------------------------------------------------------------------

# 11. Write a CI/CD pipeline for a Java application deploying to Kubernetes for Dev and Prod environments.

### Answer

I would structure the pipeline so that the application is built once,
tested once, and the same container image is promoted from Dev to Prod.

Example using GitHub Actions:

``` yaml
name: Java Application CI/CD

on:
  push:
    branches:
      - main
  pull_request:

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: java-app
  EKS_CLUSTER: production-eks

jobs:

  # -----------------------------
  # CI
  # -----------------------------
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - name: Build and Test
        run: mvn clean verify

      - name: Build Docker Image
        run: |
          docker build \
            -t ${{ env.ECR_REPOSITORY }}:${{ github.sha }} .

      - name: Image Scan
        run: |
          # Run Trivy or another approved container scanner
          trivy image \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            ${{ env.ECR_REPOSITORY }}:${{ github.sha }}

      - name: Login to AWS ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Tag and Push Image
        run: |
          docker tag \
            ${{ env.ECR_REPOSITORY }}:${{ github.sha }} \
            ${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          docker push \
            ${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

  # -----------------------------
  # DEV DEPLOYMENT
  # -----------------------------
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: dev

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig \
            --region $AWS_REGION \
            --name $EKS_CLUSTER

      - name: Deploy to Dev
        run: |
          kubectl -n dev set image deployment/java-app \
            java-app=${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          kubectl -n dev rollout status deployment/java-app --timeout=5m

      - name: Smoke Test
        run: |
          # Add application/API smoke tests here
          echo "Running Dev smoke tests"

  # -----------------------------
  # PROD DEPLOYMENT
  # -----------------------------
  deploy-prod:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ env.AWS_REGION }}
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}

      - name: Configure kubectl
        run: |
          aws eks update-kubeconfig \
            --region $AWS_REGION \
            --name $EKS_CLUSTER

      - name: Deploy to Prod
        run: |
          kubectl -n prod set image deployment/java-app \
            java-app=${{ secrets.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          kubectl -n prod rollout status deployment/java-app --timeout=5m

      - name: Verify Pods
        run: |
          kubectl -n prod get pods
          kubectl -n prod get deployment java-app
```

### How I would explain this in an interview

I would say:

> "I build and test the Java application first. Then I create one Docker
> image using the Git commit SHA and scan it before pushing it to ECR.
> After that I deploy the same image to the Dev namespace and run smoke
> tests. If Dev is healthy, the pipeline promotes the exact same image
> to Prod, normally with an approval gate. I use rolling updates and
> readiness probes so traffic only goes to healthy pods. After
> production deployment I verify rollout status, pod health, application
> logs and key metrics. If there is an issue, I rollback to the previous
> image."

### Production improvements

For a larger production setup, I would normally avoid putting all
deployment logic directly into the application pipeline. I may use:

``` text
GitHub
   |
   v
CI
   |
   +--> Maven Test
   +--> SAST / SonarQube
   +--> Docker Build
   +--> Image Scan
   +--> ECR
   |
   v
GitOps Repository
   |
   v
Argo CD
   |
   +--> EKS Dev
   |
   +--> Approval
   |
   +--> EKS Prod
```

With Argo CD, Git becomes the source of truth for Kubernetes manifests,
and deployment/rollback becomes easier to audit.

------------------------------------------------------------------------

# Quick Senior-Level Points to Remember

## 3-Tier Architecture

``` text
Route53
   ↓
CloudFront/WAF
   ↓
ALB
   ↓
Application
   ↓
RDS/Aurora
```

## CI/CD

``` text
Code
 ↓
Build
 ↓
Test
 ↓
Scan
 ↓
Docker Build
 ↓
Image Scan
 ↓
ECR
 ↓
Dev
 ↓
Smoke Test
 ↓
Approval
 ↓
Prod
 ↓
Monitor / Rollback
```

## Kubernetes Deployment

``` text
Deployment
   ↓
ReplicaSet
   ↓
Pods
   ↓
Readiness Probe
   ↓
Service
   ↓
Traffic
```

## When a Pod Crashes

``` text
kubectl get pods
        ↓
kubectl describe pod
        ↓
kubectl logs
        ↓
kubectl logs --previous
        ↓
Check events
        ↓
Find root cause
        ↓
Fix manifest/application/config
        ↓
Rollout / rollback
```

## Important Interview Principle

As a senior DevOps engineer, I would avoid saying:

> "The pod crashed, so I increased the resources."

A better answer is:

> "First I check why the pod crashed. I look at the previous container
> logs, pod events, exit code and resource usage. Once I identify the
> root cause, I change the deployment configuration or application
> accordingly. If the deployment itself is bad, I rollback first and
> then investigate."

This shows a production troubleshooting approach rather than blindly
changing Kubernetes settings.
