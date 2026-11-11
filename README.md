# Demo Repository

A simple Kubernetes/OpenShift application deployment example using YAML manifests.

This repository demonstrates how to deploy an NGINX web server on an OpenShift cluster, expose it internally using a Kubernetes Service, and make it accessible externally through an OpenShift Route.

## Architecture

The deployment consists of three Kubernetes/OpenShift resources:

```text
                  External Users
                        |
                        |
                 OpenShift Route
                 (TLS - Edge)
                        |
                        |
                 Kubernetes Service
                    (Port 80)
                        |
                        |
                NGINX Deployment
                 (2 Replicas)
                        |
                 +------+------+
                 |             |
              NGINX Pod     NGINX Pod
              (Port 8080)   (Port 8080)
```

## Repository Structure

```text
demo-repo/
│
├── deployment.yaml    # NGINX application deployment
├── service.yaml       # Internal service configuration
├── route.yaml         # OpenShift external route
└── README.md          # Project documentation
```

## Prerequisites

Before deploying the application, ensure you have:

- Access to a running OpenShift cluster.
- `oc` CLI installed and configured.
- Appropriate permissions to create Deployments, Services, and Routes.
- An active project/namespace in OpenShift.

Verify your cluster connection:

```bash
oc whoami
oc cluster-info
oc project
```

## Deployment

### 1. Clone the repository

```bash
git clone https://github.com/AshishReddy/demo-repo.git

cd demo-repo
```

### 2. Select the OpenShift project

```bash
oc project <namespace>
```

Alternatively, create a new project:

```bash
oc new-project demo-project
```

### 3. Deploy the application

Apply the deployment manifest:

```bash
oc apply -f deployment.yaml
```

This creates an NGINX Deployment with two replicas.

### 4. Create the Kubernetes Service

```bash
oc apply -f service.yaml
```

The Service exposes the application internally on port 80 and forwards traffic to port 8080 on the NGINX pods.

### 5. Create the OpenShift Route

```bash
oc apply -f route.yaml
```

The Route exposes the application externally using edge TLS termination.

### Alternative: Deploy all resources together

You can also deploy all manifests with a single command:

```bash
oc apply -f .
```

## Verification

### Check deployment status

```bash
oc get deployments
```

### Check running pods

```bash
oc get pods -l app=demo-app
```

### Check the service

```bash
oc get svc demo-app-service
```

### Check the OpenShift Route

```bash
oc get route demo-app-route
```

Retrieve the application URL:

```bash
oc get route demo-app-route \
  -o jsonpath='{.spec.host}'
```

Test application accessibility:

```bash
curl -k https://$(oc get route demo-app-route -o jsonpath='{.spec.host}')
```

## Cleanup

To remove all resources created by this repository:

```bash
oc delete -f .
```

Alternatively, delete individual resources:

```bash
oc delete route demo-app-route
oc delete service demo-app-service
oc delete deployment demo-app
```

## Configuration Overview

| Resource | Configuration |
|---|---|
| Deployment | `demo-app` |
| Container Image | `bitnami/nginx` |
| Replicas | 2 |
| Container Port | 8080 |
| Service | `demo-app-service` |
| Service Port | 80 |
| Route | `demo-app-route` |
| TLS Termination | Edge |

## Purpose

This repository can be used as a basic reference for:

- Kubernetes Deployment and Service configuration.
- OpenShift Route configuration.
- Application exposure using OpenShift.
- Deploying containerized workloads using YAML manifests.
- Practicing basic Kubernetes/OpenShift administration.

## License

This repository is intended for demonstration and learning purposes.
