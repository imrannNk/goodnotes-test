## About
A simple HTTP Echo app with Docker containerization and Kubernetes deployment manifests. 

## Features

### Application:
- **HTTP Echo Server**: Returns configurable text via environment variable or CLI flag
- **Docker Support**: Containerized application with security scanning
- **Kubernetes Deployment**: Multi-environment deployment with Kustomize
- **Load Balancing**: NGINX Ingress Controller with host-based routing

### CI Pipeline:
- **GitHub Actions CI Pipeline**: GitHub Actions workflow with security scans, and KinD Cluster setup to perform Load Testing in the CI pipeline.

### Project Structure

```
.
├── http-echo/              # Go application source
│   ├── main.go             # Main application
│   ├── handlers.go         # HTTP handlers
│   ├── Dockerfile          # Container image
│   └── go.mod              # Go dependencies
├── k8s/                    # Kubernetes manifests
│   ├── base/               # Base Kustomize resources
│   └── overlays/           # Environment-specific configs
│       ├── foo/            # foo.localhost deployment
│       └── bar/            # bar.localhost deployment
├── kind_cluster.yaml       # KinD cluster configuration
└── .github/workflows/      # CI/CD pipeline
```

## Local Development, Build and Run
Steps for local development:

<details>
<summary>Click to expand</summary>

### Prerequisites

- Go 1.21+
- Docker
- kubectl
- KinD (Kubernetes in Docker)

### Build & Run Locally

```bash
# Build the application
cd http-echo
go build -o http-echo

# Run with custom text
./http-echo -text "Hello World" -listen ":8080"

# Or using environment variable
ECHO_TEXT="Hello World" ./http-echo

# Test the endpoints
curl http://localhost:5678
curl http://localhost:5678/health
```

### Docker Build

```bash
# Build binary for Linux
cd http-echo
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o dist/linux/amd64/http-echo

# Build Docker image
docker build -t http-echo:local --build-arg BIN_NAME=http-echo .

# Run container
docker run --rm -p 5678:5678 -e ECHO_TEXT="Docker Hello" http-echo:local
```
</details>


## Kubernetes Deployment
Local KinD Cluster setup and deployment:

<details>
<summary>Click to expand</summary>

### Local KinD Cluster Setup

```bash
# Create KinD cluster
kind create cluster --config kind_cluster.yaml --name test-cluster

# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=300s

# Load Docker image to cluster
docker build -t http-echo:local --build-arg BIN_NAME=http-echo ./http-echo
kind load docker-image http-echo:local --name test-cluster

# Update manifests to use local image
sed -i 's|image: http-echo:latest|image: http-echo:local|g' k8s/base/deployment.yaml
sed -i 's|imagePullPolicy: IfNotPresent|imagePullPolicy: Never|g' k8s/base/deployment.yaml
```
### Deploy Applications (Foo and Bar)

```bash
# Deploy foo application
kubectl apply -k k8s/overlays/foo/
kubectl wait --for=condition=available --timeout=300s deployment/foo-http-echo

# Deploy bar application
kubectl apply -k k8s/overlays/bar/
kubectl wait --for=condition=available --timeout=300s deployment/bar-http-echo

# Verify deployments
kubectl get pods,svc,ingress
```

### Test Applications (connectivity testing)

```bash
# Add hosts to /etc/hosts
echo "127.0.0.1 foo.localhost bar.localhost" | sudo tee -a /etc/hosts

# Test endpoints
curl http://foo.localhost      # Returns "foo"
curl http://bar.localhost      # Returns "bar"

# Health checks
curl http://foo.localhost/health
curl http://bar.localhost/health
```

### Load Testing (local load testing using hey)
Testing 1000 requests, with concurrency level as 50
```bash
# Install hey
wget -O hey https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
chmod +x hey && sudo mv hey /usr/local/bin/

# Run load tests
hey -n 1000 -c 50 -t 30 http://foo.localhost
hey -n 1000 -c 50 -t 30 http://bar.localhost
```

</details>

## CI Pipeline Setup

### GitHub Actions Workflow

The CI pipeline (`.github/workflows/ci.yml`) runs on every pull request and includes:

#### Security Scanning
- **SAST**: Gosec security scanner for Go code
- **SCA**: Nancy scanner for dependency vulnerabilities
- **Container Scan**: Trivy vulnerability scanner for Docker images
- **K8s Security**: Trivy configuration scanner for Kubernetes manifests

#### Build & Test
- Go application build with Linux binary
- Docker image creation (local, no registry push)
- Security scanning of built image

#### Kubernetes Testing
- KinD cluster creation with ingress support
- NGINX Ingress Controller installation
- Deployment of both foo and bar applications
- Health check verification

#### Load Testing
- Performance testing with `hey` tool
- Tests both foo.localhost and bar.localhost
- Metrics collection: Average, P90, P95, Success Rate, Total Requests

#### PR Comments
- Automated PR comments with:
  - Load test results comparison table
  - Security scan status
  - Deployment verification
  - Full test output in collapsible section

### Workflow Triggers

```yaml
on:
  pull_request:
    branches: [ main, master ]
```

### Example PR Comment Output

```markdown
## CI Pipeline Results

### Load Test Results

| Metric                 | foo.localhost | bar.localhost |
|------------------------|---------------|---------------|
| Average Response Time  | 51.3ms        | 48.1ms        |
| Slowest Response Time  | 209.8ms       | 204.3ms       |
| Fastest Response Time  | 0.5ms         | 0.8ms         |
| P90 Response Time      | 99.9ms        | 99.0ms        |
| P95 Response Time      | 103.5ms       | 102.6ms       |
| Success Rate           | 100%          | 100%          |
| Total Requests         | 1000          | 1000          |

### Security Scans
- ✅ SAST scan completed
- ✅ SCA scan completed  
- ✅ Container vulnerability scan completed
- ✅ Kubernetes security scan completed
```

## Architecture

### Application Components
- **HTTP Server**: Go-based echo server with configurable responses
- **Health Endpoint**: Kubernetes-ready health checks
- **Multi-Environment**: Separate deployments for different configurations

### Kubernetes
- **Deployments**: Separate pods for foo and bar environments
- **Services**: ClusterIP services for internal communication
- **Ingress**: Host-based routing (foo.localhost, bar.localhost)
- **ConfigMaps**: Environment-specific configurations via Kustomize
- **Kustomize**: Used to avoid configuration repetition between foo and bar deployments

### Security Features
- Distroless base image for minimal attack surface
- Non-root container execution
- Resource limits and health probes
- Comprehensive security scanning in CI

## To Do's
- Add unit tests for HTTP handlers
- Add code coverage reporting
- Implement horizontal pod autoscaling (HPA)
- Fix identified CVEs from security scans (SCA, SAST, Container and K8s scans)
- Push container images to a registry in CI
- Add feature to upload SARIF file to GH Security
- Add contribution guide
