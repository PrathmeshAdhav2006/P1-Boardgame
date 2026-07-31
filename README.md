# BoardGame - End-to-End DevOps CI/CD Pipeline

A production-style DevOps pipeline for a Java (Spring Boot) Board Game Database application — covering the complete software delivery lifecycle from source code to a live, monitored deployment on Kubernetes.

---

## 📋 Project Overview

This project simulates a real-world enterprise DevOps workflow, integrating Continuous Integration, Continuous Deployment, security scanning, artifact management, container orchestration, and full-stack observability.

**Application:** Spring Boot Board Game Database (Java, Maven)

---

## 🏗️ Architecture

```
Developer → GitHub → Jenkins Pipeline → SonarQube (Quality Gate)
                            ↓
                    Nexus (Artifact Repo)
                            ↓
                  Docker Build → Trivy Scan → Docker Hub
                            ↓
              Kubernetes Cluster (kubeadm, AWS EC2)
                            ↓
         Prometheus + Grafana + Blackbox Exporter (Monitoring)
```

**Infrastructure:** AWS EC2 instances running:
- 1x Kubernetes Control Plane
- 2x Kubernetes Worker Nodes
- 1x Jenkins + Monitoring (Prometheus/Grafana) server
- 1x SonarQube + Nexus server

---

## 🔧 Tech Stack

| Category | Tools |
|---|---|
| CI/CD Orchestration | Jenkins |
| Source Control | Git / GitHub |
| Build Tool | Maven |
| Code Quality | SonarQube |
| Security Scanning | Trivy (filesystem + image scans) |
| Artifact Repository | Nexus Repository Manager |
| Containerization | Docker |
| Container Registry | Docker Hub |
| Orchestration | Kubernetes (kubeadm) |
| Infrastructure | AWS EC2 |
| Monitoring | Prometheus, Node Exporter, Blackbox Exporter |
| Visualization | Grafana |
| Notifications | Jenkins Email Extension Plugin |

---

## 🚀 CI/CD Pipeline Stages

The Jenkins pipeline is fully declarative and executes the following stages in sequence:

1. **Git Checkout** — Pulls latest source from GitHub
2. **Compile** — Compiles the Java source using Maven
3. **Test** — Runs unit tests (JUnit)
4. **Scan File System** — Trivy scans the source directory for vulnerabilities and secrets
5. **SonarQube Analysis** — Static code analysis for bugs, code smells, and security hotspots
6. **Quality Gate** — Pipeline waits for SonarQube's quality gate verdict before proceeding
7. **Maven Build** — Packages the application into a deployable JAR
8. **Publish to Nexus** — Uploads the build artifact to the Nexus snapshot repository
9. **Build and Tag Docker Image** — Builds a Docker image of the application
10. **Scan Docker Image** — Trivy scans the built image for OS and dependency vulnerabilities
11. **Push Docker Image** — Pushes the tagged image to Docker Hub
12. **Deploy to Kubernetes** — Applies Kubernetes manifests (namespace, deployment, service) to the cluster
13. **Verify Deployment** — Confirms pods and services are running as expected
14. **Post Actions** — Sends an automated email notification (build status + attached Trivy report) regardless of pipeline outcome

---

## ☸️ Kubernetes Deployment

- **Cluster type:** Self-managed via `kubeadm` on AWS EC2 (1 control plane + 2 worker nodes)
- **Namespace:** `webapp`
- **Deployment:** 3 replicas of the application pod
- **Service type:** LoadBalancer (exposed via NodePort in the absence of a cloud load balancer)
- **Security:** A dedicated Jenkins Kubernetes Service Account is used for deployments, scoped to the `webapp` namespace via a custom **Role** and **RoleBinding** — granting only the permissions needed to manage deployments, services, and related resources (least-privilege access, not cluster-admin).

```bash
kubectl get all -n webapp
```
```
NAME                                          READY   STATUS    RESTARTS   AGE
pod/boardgame-deployment-xxxxx                1/1     Running   0          8m
pod/boardgame-deployment-xxxxx                1/1     Running   0          8m
pod/boardgame-deployment-xxxxx                1/1     Running   0          8m

NAME                        TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)
service/boardgame-service   LoadBalancer   10.107.43.205   <pending>     80:30379/TCP

NAME                                     READY   UP-TO-DATE   AVAILABLE
deployment.apps/boardgame-deployment     3/3     3            3
```

---

## 🔐 Security & Quality Gates

- **SonarQube Quality Gate** must pass before the pipeline proceeds to build/deploy — blocking releases that don't meet code quality standards.
- **Trivy** scans run at two points: against the source filesystem (dependencies, secrets) and again against the final Docker image (OS packages, libraries) — catching vulnerabilities introduced at either the code or container layer.
- **RBAC-scoped access** ensures Jenkins can only act within its designated namespace, rather than holding broad cluster permissions.

---

## 📊 Monitoring & Observability

| Component | Purpose | Port |
|---|---|---|
| Prometheus | Metrics collection & storage | 9090 |
| Node Exporter | Host-level metrics (CPU, RAM, disk, network) per worker node | 9100 |
| Blackbox Exporter | External HTTP probing — uptime, response time, status code | 9115 |
| Grafana | Dashboards for visualizing all of the above | 3000 |

**Dashboards implemented:**
- **Node Exporter Full** — real-time CPU, memory, disk, and network usage per node
- **Prometheus Blackbox Exporter** — live uptime status, HTTP response code, and probe duration for the deployed application

---

## 📧 Notifications

Every pipeline run sends an automated email with:
- Build number and pass/fail status (color-coded banner: green = success, red = failure)
- Link to the Jenkins console output
- Attached Trivy vulnerability scan report (image-level)

---

## 📈 Key Learnings

Beyond following a written procedure, this project involved diagnosing and resolving real infrastructure issues, including:
- AWS EC2 vCPU service quota limits when provisioning multiple instances
- Kubernetes RBAC misconfigurations (Role/RoleBinding namespace mismatches) blocking Jenkins deployments
- Docker daemon socket permission errors preventing Jenkins from building images
- SonarQube Scanner CLI / JVM compatibility issues
- YAML indentation errors in Prometheus scrape configurations
- Structuring least-privilege Kubernetes access for CI/CD automation instead of using cluster-admin

---

## 🗂️ Repository Structure

```
.
├── src/                    # Application source code
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── Jenkinsfile             # Declarative CI/CD pipeline definition
├── pom.xml                 # Maven project configuration
└── README.md
```

---

## 🔗 Related Links

- **Docker Hub Image:** `prathmeshadhav2006/boardgame:latest`
- **SonarQube Project:** BoardGame
- **Nexus Artifact Path:** `com/javaproject/database_service_project`

---

*This project was built as a hands-on exercise in end-to-end DevOps practices — CI/CD automation, container security, Kubernetes deployment, and infrastructure observability.*
