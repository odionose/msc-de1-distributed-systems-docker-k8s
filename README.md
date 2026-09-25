# Docker & Local Kubernetes Project

## 1. Project Overview

This project containerizes and deploys a small Flask REST API using Docker and a local Kubernetes cluster.

The application is based on the original UBC Flask Sample App - https://github.com/ubc/flask-sample-app

Docker image repository: https://hub.docker.com/r/nathanaelodion/msc-de1-flask-app

The application provides a simple item API:

- `GET /` — returns a welcome message
- `GET /items` — returns stored items
- `GET /items/<id>` — returns one item
- `POST /items` — adds an item

The project demonstrates containerization, container security, image scanning, Docker Hub publishing, Kubernetes deployment, service discovery, scaling, self-healing, rolling updates, rollback, and network isolation.

## Architecture

```text
                         Docker Hub
                             |
                             v
              nathanaelodion/msc-de1-flask-app:1.0.2
                             |
                             v
                 +-----------------------+
                 |   kind Kubernetes     |
                 |   msc-de1-cluster     |
                 +-----------------------+
                   |        |          |
                   v        v          v
              Control     Worker     Worker2
              Plane                   |
                                      |
                         +------------+------------+
                         | Service: flask-app      |
                         | ClusterIP :5000         |
                         +------------+------------+
                                      |
                         +------------+------------+
                         |            |            |
                         v            v            v
                      Flask Pod   Flask Pod    Flask Pod
                      Replica 1   Replica 2    Replica 3
```

## 2. Requirements

Install:

- Docker
- Docker Compose
- kubectl
- kind
- Git

## 3. Run the Application Locally

Clone the repository:

```bash
git clone <repository-url>
cd msc-de1-distributed-systems-docker-k8s
```

Install the Python dependency:

```bash
pip install -r requirements.txt
```

Run the application:

```bash
PORT=5001 python run.py
```

Test it:

```bash
curl http://localhost:5001/
curl http://localhost:5001/items
```

Run the tests:

```bash
pytest
```

## 4. Run with Docker

Build the image:

```bash
docker build -t msc-de1-flask-app:1.0.2 .
```

Run it:

```bash
docker run -d \
  --name flask-app \
  -p 5000:5000 \
  msc-de1-flask-app:1.0.2
```

Test:

```bash
curl http://localhost:5000/
```

Stop and remove the container:

```bash
docker rm -f flask-app
```

## 5. Run with Docker Compose

```bash
docker compose up --build
```

Test:

```bash
curl http://localhost:5001/
```

Stop:

```bash
docker compose down
```

## 6. Security

The Docker image runs as a non-root user and includes a container health check.

The Compose configuration also uses:

- `no-new-privileges`
- dropped Linux capabilities
- read-only root filesystem

Security scan results and the SBOM are stored in:

- `security/`

## 7. Reproduce the Kubernetes Deployment

Create the three-node kind cluster:

```bash
kind create cluster \
  --name msc-de1-cluster \
  --config kind/kind-config.yaml
```

Apply the Kubernetes resources:

```bash
kubectl apply -f k8s/
```

Check the deployment:

```bash
kubectl get nodes
kubectl get pods -n flask-app
kubectl get service -n flask-app
```

The deployment runs three replicas.

Test the application from inside the cluster:

```bash
kubectl run curl-test \
  -n flask-app \
  --rm -it \
  --image=curlimages/curl \
  -- curl http://flask-app:5000/
```

The Kubernetes configuration includes:

- non-root execution
- dropped capabilities
- read-only root filesystem
- seccomp
- resource requests and limits
- readiness and liveness probes
- ClusterIP service
- NetworkPolicy

## 8. Reproduce Kubernetes Behaviour

Scale:

```bash
kubectl scale deployment flask-app -n flask-app --replicas=3
```

Test self-healing:

```bash
kubectl delete pod <pod-name> -n flask-app
kubectl get pods -n flask-app -w
```

Test a rolling update:

```bash
kubectl patch deployment flask-app -n flask-app \
  -p '{"spec":{"template":{"metadata":{"annotations":{"rollout-demo":"v1"}}}}}'

kubectl rollout status deployment/flask-app -n flask-app
```

View rollout history:

```bash
kubectl rollout history deployment/flask-app -n flask-app
```

Rollback:

```bash
kubectl rollout undo deployment/flask-app -n flask-app --to-revision=1
```

## 9. Evidence

Project evidence is organised in:

- `evidence/`

Kubernetes manifests and cluster configuration are in:

- `k8s/`
- `kind/`

Security files are in:

- `security/`

## 10. Cleanup

Remove the Kubernetes cluster:

```bash
kind delete cluster --name msc-de1-cluster
```

## 11. Limitations and Technical Considerations

The application uses process-local in-memory state, meaning each Kubernetes replica maintains its own items list. The Service distributes requests between replicas but does not synchronize their state. A production implementation would therefore require shared persistent storage, such as a database.

The Kubernetes environment was created with kind, so although it demonstrates real Kubernetes scheduling, networking, scaling and recovery behaviour, the nodes run within a local development environment rather than production infrastructure. Production deployment would require additional considerations such as persistent storage, monitoring, load balancing, backups and secret management.

The final Docker image contained no Critical vulnerabilities, although High, Medium and Low findings remained. The remaining higher-severity findings were associated with base-image components without fixed versions reported by the scanner. The project therefore demonstrates vulnerability reduction and risk documentation rather than claiming a completely vulnerability-free image.


## 12. License

This project follows the license included in the repository.

The original Flask Sample App is provided by UBC:

https://github.com/ubc/flask-sample-app