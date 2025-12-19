# OpenStudyBuilder Helm Charts

This directory contains Helm charts for deploying OpenStudyBuilder to Kubernetes clusters.

## Status quo of the OpenStudyBuilder

The OpenStudyBuilder solution introduces a new approach for working with studies that once fully implemented will drive end-to-end consistency and more efficient processes - all the way from protocol development and CRF design - to creation of datasets, analysis, reporting, submission to health authorities and public disclosure of study information.

OpenStudyBuilder is a next generation end-to-end clinical data standards and study specification solution, enabling clinical study data solutions to use linked metadata for higher degree of automation, limiting manual document driven work processes, enabling a Digital Data Flow approach.

OpenStudyBuilder is the open source version of the internal StudyBuilder solution at Novo Nordisk. Not all titles or logos in the application are yet changed to be 'OpenStudyBuilder' - when the term 'StudyBuilder' is used, it is therefore a synonym for 'OpenStudyBuilder'. This will be changed in coming updates.

For further information on the OpenStudyBuilder solution, please refer to the [OpenStudyBuilder homepage](https://openstudybuilder.com).

## Overview

The Helm charts in this directory provide a streamlined way to deploy and manage OpenStudyBuilder components in Kubernetes environments.

**NOTE**: The chart version is mapped to a unique OpenStudyBuilder version, not latest (see `Chart.yaml` file, values `version` and `appVersion` respectively). In production it is recommended to define your `customImages` for each running service, hence avoiding upgrading Helm chart version with a new OpenStudyBuilder version too.

The following services are included in the OpenStudyBuilder Helm chart:

- **database** (Neo4j graph database with initial data if demo flag enabled, see next section `Neo4j Deployment Modes`)
- **api** (FastAPI backend application)
- **consumerapi** (FastAPI application for additional integrations)
- **neodash** (NeoDash dashboards for OpenStudyBuilder)
- **frontend (sb)** (Nginx hosting Vue.js StudyBuilder UI)
- **documentation (docs)** (Nginx hosting Vue.js documentation portal)

### Neo4j Deployment Modes

OpenStudyBuilder Helm supports three Neo4j deployment options:

- Subchart: `neo4j.enabled: true` — uses the official Neo4j Helm subchart (recommended for production). This requires importing data manually (restoring or running the tooling jobs)
- Demo: `neo4j.demo: true` — deploys a lightweight demo with sample data using the public image (recommended by default for quick local trials).
- External: when defining both previous settings to `false` means the DB is managed somewhere else. One will only require setting the right connection string values and credentials where apply.

By default, the chart is configured for the demo mode with sample data:

- `neo4j.demo: true`
- `neo4j.enabled: false`

**Important**: This default aims to provide an immediate, ready-to-use experience. The demo Neo4j Deployment and Service start with demo defaults and do not require storage class provisioning. Do not use this setup in production.

To switch to the production-ready subchart:

- Set `neo4j.demo: false`
- Set `neo4j.enabled: true`

Do not enable both at the same time. The chart includes a guard that fails fast if both are true, guiding you to choose one mode.

## System Requirements

A Kubernetes cluster with at least 6GB of memory allocated is required for running all OpenStudyBuilder components (demo setup). For production ready deployments at least 15GB are required (depending on expected system load).

The solution has been tested on the following platforms:

- **kind** (Kubernetes in Docker) - for local development
- **OpenShift** - for enterprise deployments
- Standard Kubernetes clusters (1.19+)

### Resource Requirements

The default resource allocations are defined in `values.yaml`. Please modify them accordingly to your deployment needs.

##  Helm Deployment Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure (if persistence is enabled)

## Local Development with KinD

For local development and testing, you can use [KinD](https://kind.sigs.k8s.io/) (Kubernetes in Docker) to run the chart on your local machine.

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [KinD](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Helm 3.0+](https://helm.sh/docs/intro/install/)

### Setup kind Cluster

1. **Create a kind cluster**:

   ```bash
   kind create cluster --name kind-cluster # or a default kind kubernetes cluster if using Docker Desktop
   ```

2. **Verify the cluster is running**:

   ```bash
   kubectl cluster-info --context kind-kind-cluster # or kubectl cluster-info --context docker-desktop if using Docker Desktop
   ```

### Install the Chart

1. **Switch to the kind context** (if not already):

   ```bash
   kubectl config use-context kind-kind-cluster # or kubectl config use-context docker-desktop if using Docker Desktop
   ```

2. **Install the chart dependencies**:

   ```bash
   # From the charts directory
   cd charts
   helm dependency update ./openstudybuilder
   ```

3. **Install the chart**:

   ```bash
   # From the charts directory
   helm upgrade --install my-osb ./openstudybuilder --rollback-on-failure (or --atomic flag depending on your Helm version)
   ```

4. **Monitor the deployment**:

   ```bash
   kubectl get pods -w
   ```

   Wait for all pods to be in `Running` state and ready.

### Accessing the Application

To access the services running in the kind cluster, you need to set up port-forwarding:

1. **Set up port-forwarding for the frontend service and Neo4j bolt service**:

   In a terminal:

   ```bash
   kubectl port-forward service/sb 5005:5005  # required to access the UI
   ```

   In another terminal:

   ```bash
   kubectl port-forward service/mdrdb 5002:bolt  # required to access neodash
   ```

   Leave this running in the terminal. Open new terminal windows for additional services.

2. **Neo4j client (Optional)** (in separate terminals):

   ```bash
   # Neo4j HTTP
   kubectl port-forward service/mdrdb 7474:7474
   ```

Once the port-forwarding is set up, you can access the OpenStudyBuilder services:

- **StudyBuilder main application**: <http://localhost:5005>

- **StudyBuilder documentation**: <http://localhost:5005/doc> (also accessible from Studybuilder UI)

- **StudyBuilder API (backend application)**: <http://localhost:5005/api/docs>

- **StudyBuilder Consumer API (backend application)**: <http://localhost:5005/consumer-api/docs>

- **Neo4j dashboard web client**: <http://localhost:5005/neodash> (also accessible from Studybuilder UI)

- **Neo4j database web client**: <http://localhost:7474/browser>

  The default username is `neo4j` and the default password is `changeme1234`

### Stopping the Services

To stop the port-forwarding, simply press `Ctrl+C` in each terminal where port-forwarding is running.

The Helm release and pods will continue running in the Kubernetes cluster. To stop the pods:

```bash
# Scale down deployments (keeps configuration but stops pods)
kubectl scale deployment --all --replicas=0

# Or uninstall the release completely
helm uninstall my-osb
```

### Upgrading the Chart

To apply changes after modifying values or templates:

```bash
helm upgrade --install my-osb ./openstudybuilder --rollback-on-failure (or --atomic flag depending on your Helm version)
```

### Cleaning up the Kubernetes Environment

To completely clean up the OpenStudyBuilder deployment:

1. **Uninstall the Helm release**:

   ```bash
   helm uninstall my-osb
   ```

2. **Delete remaining PVCs** (if any):

   Persistent Volume Claims may be retained depending on the retention policy:

   ```bash
   # List all PVCs
   kubectl get pvc

   # Delete specific PVCs
   kubectl delete pvc <pvc-name>

   # Or delete all PVCs in the namespace
   kubectl delete pvc --all
   ```

3. **Delete the kind cluster** (for local development):

   ```bash
   kind delete cluster --name kind-cluster
   ```

4. **Clean up Docker resources** (optional, for kind clusters):

   ```bash
   # Remove unused Docker images
   docker image prune -a

   # Remove unused volumes
   docker volume prune
   ```
   kind delete cluster --name kind-cluster
   ```

### Troubleshooting

- **Check pod logs**:
  ```bash
  kubectl logs <pod-name>
  kubectl logs <pod-name> -f  # Follow logs
  ```

- **Describe pod for events**:
  ```bash
  kubectl describe pod <pod-name>
  ```

- **Access pod shell**:
  ```bash
  kubectl exec -it <pod-name> -- /bin/bash
  ```

- **Check service endpoints**:
  ```bash
  kubectl get endpoints
  ```

- **View all resources**:
  ```bash
  kubectl get all
  ```

## Installing the Chart

### From Local Source

To install the chart from local source with the release name `my-osb`:

```bash
# From the charts directory
helm install my-osb ./openstudybuilder

# With custom values
helm install my-osb ./openstudybuilder -f custom-values.yaml
```

### From Published Chart Repository

Once the chart is published, you can install it directly from the Helm repository:

```bash
# Add the OpenStudyBuilder Helm repository
helm repo add openstudybuilder https://novoNordisk-OpenSource.github.io/openstudybuilder-accelerators

# Update your local Helm chart repository cache
helm repo update

# List available versions
helm search repo openstudybuilder/openstudybuilder --versions

# Install the latest chart
helm install my-osb openstudybuilder/openstudybuilder

# Or install a specific version
helm install my-osb openstudybuilder/openstudybuilder --version 0.1.0
```

### Listing Available Versions

To see all available versions of the chart:

```bash
# List all versions from the repository
helm search repo openstudybuilder/openstudybuilder --versions

# Search with output in YAML format for more details
helm search repo openstudybuilder/openstudybuilder --versions -o yaml

# List versions with additional information (JSON format)
helm search repo openstudybuilder/openstudybuilder --versions -o json
```

For OCI registry installations:

```bash
# List versions from OCI registry (requires helm 3.8+)
helm show chart oci://ghcr.io/novoNordisk-OpenSource/openstudybuilder

# Or use registry API/tools to browse available tags
```

## Uninstalling the Chart

To uninstall/delete the `my-osb` deployment:

```bash
helm uninstall my-osb
```

## Configuration

All parameters are defined in `values.yaml` and provided with good defaults that provide a working demo environment.

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`:

```bash
helm install my-osb ./openstudybuilder \
  --set replicaCount=2 \
  --set ingress.enabled=true
```

Alternatively, recommended for production, a YAML file that specifies the values for the parameters can be provided externally:

```bash
helm install my-osb ./openstudybuilder -f values.yaml
```

## Examples

### Enable Ingress

```yaml
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: osb.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: osb-tls
      hosts:
        - osb.example.com
```

### Enable Persistence

```yaml
persistence:
  enabled: true
  storageClass: "standard"
  size: 20Gi
```

### Set Resource Limits

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

### Enable Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
```

## Publishing the Chart

### Using GitHub Pages

This repository can use GitHub Pages to host the Helm chart repository. Here's how to publish:

#### 1. Package the Chart

First, package the Helm chart:

```bash
# From the repository root
helm package charts/openstudybuilder -d packages/
```

This creates a `.tgz` file in the `packages/` directory.

#### 2. Generate the Repository Index

Create or update the Helm repository index:

```bash
# Generate index.yaml
helm repo index packages/ --url https://novoNordisk-OpenSource.github.io/openstudybuilder-accelerators
```

#### 3. Commit and Push

```bash
git add packages/
git commit -m "Release Helm chart version X.Y.Z"
git push origin main
```

#### 4. Enable GitHub Pages

1. Go to your repository settings on GitHub
2. Navigate to **Pages** section
3. Set the source to the `main` branch and `/packages` folder (or configure accordingly)
4. GitHub will provide the URL where your charts are hosted

### Using GitHub Actions (Automated)

You can automate chart publishing with GitHub Actions. Create `.github/workflows/release-chart.yml`:

```yaml
name: Release Helm Chart

on:
  push:
    branches:
      - main
    paths:
      - 'charts/**'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Configure Git
        run: |
          git config user.name "$GITHUB_ACTOR"
          git config user.email "$GITHUB_ACTOR@users.noreply.github.com"

      - name: Install Helm
        uses: azure/setup-helm@v3

      - name: Run chart-releaser
        uses: helm/chart-releaser-action@v1.5.0
        env:
          CR_TOKEN: "${{ secrets.GITHUB_TOKEN }}"
```

This workflow automatically packages and publishes charts when changes are pushed to the `charts/` directory.

### Using OCI Registry (Alternative)

Helm 3 supports OCI registries for chart storage:

```bash
# Login to registry (e.g., GitHub Container Registry)
echo $GITHUB_TOKEN | helm registry login ghcr.io -u USERNAME --password-stdin

# Package and push
helm package charts/openstudybuilder
helm push openstudybuilder-0.1.0.tgz oci://ghcr.io/novoNordisk-OpenSource

# Install from OCI registry
helm install my-osb oci://ghcr.io/novoNordisk-OpenSource/openstudybuilder --version 0.1.0
```

## Upgrading the Chart

To upgrade an existing release with new values:

```bash
helm upgrade my-osb openstudybuilder/openstudybuilder -f custom-values.yaml
```

To upgrade to a specific version:

```bash
helm upgrade my-osb openstudybuilder/openstudybuilder --version 0.2.0
```

## Support

For issues and questions, please refer to the main [OpenStudyBuilder repository](https://github.com/NovoNordisk-OpenSource/openstudybuilder-solution).
