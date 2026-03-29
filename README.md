# vprofile-action — GitOps CI/CD Pipeline with GitHub Actions + EKS

A complete GitOps continuous delivery pipeline for the VProfile multi-tier Java application. On every code push, GitHub Actions automatically builds a Docker image, pushes it to Amazon ECR, and deploys it to an AWS EKS Kubernetes cluster — no manual steps required.

---

## Pipeline Architecture

```
Developer pushes code
         │
         ▼
  GitHub Actions triggered
         │
    ┌────┴────┐
    │  Stage 1 │  Maven Build + Unit Tests
    └────┬────┘
         │
    ┌────▼────┐
    │  Stage 2 │  SonarQube Code Analysis + Quality Gate
    └────┬────┘
         │
    ┌────▼────┐
    │  Stage 3 │  Docker Build + Push to Amazon ECR
    └────┬────┘
         │
    ┌────▼────┐
    │  Stage 4 │  kubectl apply → Deploy to EKS
    └────┬────┘
         │
         ▼
  App live on Kubernetes
  (Accessible via NGINX Ingress)
```

---

## Repository Structure

```
vprofile-action/
├── .github/
│   └── workflows/
│       ├── main.yml          # Full CI/CD pipeline
│       └── codeql.yml        # Security scanning
├── kubernetes/
│   ├── appdeploy.yaml        # Tomcat app Deployment
│   ├── appservice.yaml       # App ClusterIP Service
│   ├── appingress.yaml       # NGINX Ingress (host-based routing)
│   ├── dbdeploy.yaml         # MySQL Deployment
│   ├── dbservice.yaml        # DB ClusterIP Service
│   ├── dbpvc.yaml            # PersistentVolumeClaim for MySQL
│   ├── mcdep.yaml            # Memcached Deployment
│   ├── mcservice.yaml        # Memcached Service
│   ├── rmqdeploy.yaml        # RabbitMQ Deployment
│   ├── rmqservice.yaml       # RabbitMQ Service
│   └── secret.yaml           # K8s Secret (DB credentials)
├── ansible/                  # Tomcat provisioning playbooks
├── Dockerfile                # Multi-stage app image
├── Jenkinsfile               # Alternative Jenkins pipeline
└── pom.xml                   # Maven build config
```

---

## GitHub Actions Workflow

### Workflow Triggers
```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
```

### Pipeline Stages

**1. Build & Test**
```
mvn clean install    → Compile + package WAR
mvn test             → Unit test suite
mvn verify           → Integration tests
mvn checkstyle:check → Code style validation
```

**2. Code Quality (SonarQube)**
```
sonar-scanner  → Static analysis
               → Quality Gate check (pipeline fails if gate fails)
```

**3. Docker Build & Push to ECR**
```
docker build -t vprofileapp .
aws ecr get-login-password | docker login
docker tag  vprofileapp <account>.dkr.ecr.<region>.amazonaws.com/vprofileapp:<tag>
docker push <account>.dkr.ecr.<region>.amazonaws.com/vprofileapp:<tag>
```

**4. Deploy to EKS**
```
aws eks update-kubeconfig --name <cluster>
kubectl apply -f kubernetes/
kubectl rollout status deployment/vproapp
```

---

## Kubernetes Resources

| Resource | Kind | Purpose |
|----------|------|---------|
| `vproapp` | Deployment | Tomcat app (pulls from ECR) |
| `vproapp-service` | Service | Internal ClusterIP for app |
| `vpro-ingress` | Ingress | NGINX Ingress for external access |
| `vprodb` | Deployment | MySQL with init container |
| `vprodb-pvc` | PersistentVolumeClaim | Persistent MySQL data volume |
| `vprocache01` | Deployment | Memcached cache layer |
| `vpromq01` | Deployment | RabbitMQ message broker |
| `app-secret` | Secret | DB credentials (base64 encoded) |

### Init Container Pattern
The app Deployment uses init containers to prevent startup race conditions:
```yaml
initContainers:
  - name: init-mydb        # Waits for MySQL DNS to resolve
  - name: init-memcache    # Waits for Memcached DNS to resolve
```

### Ingress Configuration
```yaml
spec:
  ingressClassName: nginx
  rules:
  - host: vprofile.yourdomain.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: vproapp-service
            port: 8080
```

---

## GitHub Secrets Required

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS IAM access key |
| `AWS_SECRET_ACCESS_KEY` | AWS IAM secret key |
| `AWS_REGION` | Target AWS region |
| `ECR_REGISTRY` | ECR registry URL |
| `EKS_CLUSTER` | EKS cluster name |
| `SONAR_TOKEN` | SonarQube authentication token |
| `SONAR_URL` | SonarQube server URL |

---

## Key Skills Demonstrated

- GitHub Actions CI/CD pipeline (build → test → scan → build image → deploy)
- Docker multi-stage builds for optimised container images
- Amazon ECR image registry management
- Kubernetes manifest authoring (Deployments, Services, Ingress, PVC, Secrets)
- NGINX Ingress controller configuration with host-based routing
- Init containers for dependency readiness enforcement
- GitOps workflow — Git as single source of truth for cluster state
- SonarQube quality gate integration in CI pipeline
- AWS EKS deployment via kubectl in a CI environment
