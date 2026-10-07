# DevOps Interview Questions & Answers

A comprehensive collection of DevOps, Kubernetes, Docker, and CI/CD interview questions with detailed answers.

---

## Table of Contents

1. [Design a 3-tier architecture](#1-design-a-3-tier-architecture-explain-the-workflow-step-by-step)
2. [How does CI/CD work? Maven build commands](#2-how-does-cicd-work-what-commands-do-you-run-during-a-maven-build)
3. [How does GitHub work?](#3-how-does-github-work-how-do-you-clone-pull-and-commit-code)
4. [What is PDB (Pod Disruption Budget)?](#4-what-is-pdb-pod-disruption-budget)
5. [Readiness and Liveness Probes](#5-what-are-readiness-and-liveness-probes-what-do-you-add-in-deploymentyaml-for-these-share-the-syntax)
6. [Practical steps after CI/CD build](#6-after-building-an-application-in-cicd-what-practical-steps-do-you-perform-next)
7. [Deployment YAML manifest](#7-write-a-deploymentyaml-manifest)
8. [Multi-stage Dockerfile](#8-write-a-multi-stage-dockerfile)
9. [Pod crashes - deployment.yaml changes](#9-if-a-pod-crashes-what-changes-would-you-make-in-deploymentyaml)
10. [ADD vs COPY in Docker](#10-what-is-the-difference-between-add-and-copy-in-docker)
11. [CI/CD pipeline for Java + Kubernetes](#11-write-a-cicd-pipeline-for-a-java-application-deploying-to-kubernetes-for-dev-and-prod-environments)

---

## 1. Design a 3-tier architecture. Explain the workflow step by step.

### 3-Tier Architecture Design

- **Presentation Tier (Tier 1):** The user interface. It runs in the user's browser or mobile app. Examples: React, Angular, Vue.js, or a mobile app.
- **Application Tier (Tier 2):** The business logic. It processes requests, performs calculations, and makes decisions. Examples: Java Spring Boot, Node.js, Python Django.
- **Data Tier (Tier 3):** The database and data storage. It stores and retrieves data. Examples: PostgreSQL, MySQL, MongoDB.

### Workflow Step by Step

1. **User Action:** A user interacts with the Presentation Tier (e.g., clicks a "Login" button).
2. **Request Sent:** The Presentation Tier sends an HTTP/HTTPS request (e.g., `POST /api/login`) to the Application Tier.
3. **Business Logic:** The Application Tier receives the request. It validates the input, applies business rules (e.g., check if the password matches), and may call other services.
4. **Data Access:** The Application Tier connects to the Data Tier to fetch or update data (e.g., `SELECT * FROM users WHERE email = ...`).
5. **Data Response:** The Data Tier returns the requested data to the Application Tier.
6. **Response Formatted:** The Application Tier formats the result (e.g., a JSON Web Token) and sends it back to the Presentation Tier.
7. **UI Update:** The Presentation Tier receives the response and updates the user interface (e.g., redirects to the dashboard).

---

## 2. How does CI/CD work? What commands do you run during a Maven build?

### How CI/CD Works

- **Continuous Integration (CI):** Developers frequently merge code into a shared repository. Each merge triggers an automated build and test process to detect integration errors early.
- **Continuous Delivery/Deployment (CD):** After CI, the code is automatically prepared for release. Continuous Delivery means it's ready to deploy manually. Continuous Deployment means it deploys automatically to production.

### Common Maven Commands during a Build

- `mvn clean`: Deletes the `target` directory to ensure a fresh build.
- `mvn compile`: Compiles the source code.
- `mvn test`: Runs unit tests.
- `mvn package`: Packages the compiled code into a JAR or WAR file.
- `mvn install`: Installs the package into the local repository for use as a dependency.
- `mvn verify`: Runs any checks to verify the package is valid.
- `mvn deploy`: Copies the final package to a remote repository.

A typical CI command sequence: `mvn clean package` or `mvn clean install`.

---

## 3. How does GitHub work? How do you clone, pull, and commit code?

### How GitHub Works

GitHub is a cloud-based platform for hosting Git repositories. It provides collaboration features like pull requests, issues, and actions. You have a local copy of the repository on your machine, and you push/pull changes to/from a remote repository on GitHub.

### Commands

- **Clone:** To copy a remote repository to your local machine for the first time.
  ```bash
  git clone https://github.com/username/repo.git
  ```

- **Pull:** To fetch changes from the remote repository and merge them into your local branch.
  ```bash
  git pull origin main
  ```

- **Commit:** To save your local changes to the local Git history.
  ```bash
  git add .
  git commit -m "Your descriptive commit message"
  git push origin main
  ```

---

## 4. What is PDB (Pod Disruption Budget)?

A **Pod Disruption Budget (PDB)** is a Kubernetes resource that limits the number of pods of a replicated application that can be down simultaneously due to voluntary disruptions (e.g., node draining for maintenance, cluster upgrades).

It ensures high availability by preventing operations that would take down too many pods at once.

### Example

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

This ensures at least 2 pods are always available.

---

## 5. What are readiness and liveness probes? What do you add in deployment.yaml for these? Share the syntax.

- **Liveness Probe:** Checks if the container is still running. If it fails, Kubernetes restarts the container.
- **Readiness Probe:** Checks if the container is ready to accept traffic. If it fails, the pod is removed from the service's endpoints.

### Syntax in `deployment.yaml`

```yaml
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
```

---

## 6. After building an application in CI/CD, what practical steps do you perform next?

1. **Run Tests:** Execute unit, integration, and security tests.
2. **Build a Container Image:** Create a Docker image from the built artifact.
3. **Scan the Image:** Use tools like Trivy or Clair to scan for vulnerabilities.
4. **Push the Image:** Push the image to a container registry (e.g., Docker Hub, ECR, GCR).
5. **Deploy to an Environment:** Update Kubernetes manifests or Helm charts with the new image tag.
6. **Apply to Kubernetes:** Run `kubectl apply -f deployment.yaml` or `helm upgrade`.
7. **Verify Deployment:** Check pod status, logs, and run smoke tests.
8. **Notify the Team:** Send a notification to Slack, Teams, or email.

---

## 7. Write a deployment.yaml manifest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: my-registry/my-app:1.0.0
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## 8. Write a multi-stage Dockerfile.

```dockerfile
# Stage 1: Build the application
FROM maven:3.8.5-openjdk-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Create the runtime image
FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=build /app/target/my-app-1.0.0.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## 9. If a pod crashes, what changes would you make in deployment.yaml?

You would investigate the crash first using `kubectl logs` and `kubectl describe pod`. Common changes to `deployment.yaml` include:

1. **Adjust Resource Limits:** Increase `resources.limits.memory` if the pod is OOMKilled.
2. **Fix Probes:** Increase `initialDelaySeconds` or adjust the probe path if the app takes longer to start.
3. **Add a Restart Policy:** Ensure `restartPolicy: Always` is set (default).
4. **Change Image:** Roll back to a previous stable image version.
5. **Add Environment Variables:** Fix missing configuration causing the crash.

### Example Change (Memory Limit)

```yaml
resources:
  limits:
    memory: "1Gi" # Increased from 512Mi
```

---

## 10. What is the difference between ADD and COPY in Docker?

| Feature | `COPY` | `ADD` |
| :--- | :--- | :--- |
| **Purpose** | Copies local files/directories into the image. | Same as COPY, but with extra features. |
| **Remote URLs** | Does not support remote URLs. | Can fetch files from a remote URL. |
| **Auto-extraction** | Does not extract archives. | Automatically extracts local tar archives. |
| **Best Practice** | **Preferred** for most use cases. | Use only when you need its specific features. |

**Recommendation:** Always use `COPY` unless you specifically need `ADD`'s auto-extraction or remote URL feature.

---

## 11. Write a CI/CD pipeline for a Java application deploying to Kubernetes for Dev and Prod environments.

Here is a **GitHub Actions** example:

```yaml
name: Java CI/CD to Kubernetes

on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

    - name: Build with Maven
      run: mvn clean package

    - name: Build Docker image
      run: docker build -t my-registry/my-app:${{ github.sha }} .

    - name: Push Docker image
      run: |
        echo ${{ secrets.DOCKER_PASSWORD }} | docker login -u ${{ secrets.DOCKER_USERNAME }} --password-stdin
        docker push my-registry/my-app:${{ github.sha }}

    - name: Deploy to Dev
      run: |
        kubectl set image deployment/my-app-deployment my-app-container=my-registry/my-app:${{ github.sha }} -n dev
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG_DEV }}

    - name: Deploy to Prod (Manual Approval)
      if: github.ref == 'refs/heads/main'
      run: |
        kubectl set image deployment/my-app-deployment my-app-container=my-registry/my-app:${{ github.sha }} -n prod
      env:
        KUBECONFIG: ${{ secrets.KUBECONFIG_PROD }}
```

This pipeline builds, tests, containerizes, and deploys to Dev automatically, and to Prod (with a manual approval gate if configured in GitHub Environments).

---

## Contributing

Feel free to fork this repository and submit pull requests for any improvements or additional questions.
