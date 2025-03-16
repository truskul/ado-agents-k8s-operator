# Azure DevOps Agent Operator

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Go Report Card](https://goreportcard.com/badge/github.com/truskul/azdevops-agent-operator)](https://goreportcard.com/report/github.com/truskul/azdevops-agent-operator)
[![Go Reference](https://pkg.go.dev/badge/github.com/truskul/azdevops-agent-operator.svg)](https://pkg.go.dev/github.com/truskul/azdevops-agent-operator)
[![Docker Pulls](https://img.shields.io/docker/pulls/truskul/azdevops-agent-operator.svg)](https://hub.docker.com/r/truskul/azdevops-agent-operator)
![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/truskul/azdevops-agent-operator/build.yml?branch=main)

A Kubernetes operator that manages Azure DevOps build and release agents as pods within your cluster. It handles agent lifecycle, scaling based on queue length, and seamless updates.

<div align="center">
  <img src="docs/images/operator-diagram.png" alt="Azure DevOps Agent Operator Diagram" width="600"/>
</div>

## Features

- **Automatic Agent Deployment**: Deploy self-hosted Azure DevOps agents in Kubernetes
- **Dynamic Scaling**: Automatically scale up/down based on job queue length
- **Agent Lifecycle Management**: Handle graceful shutdown and draining of agents
- **Multiple Pools Support**: Manage different agent pools with specific settings
- **Azure DevOps API Integration**: Register/unregister agents automatically
- **Monitoring and Metrics**: Export Prometheus metrics for agent status

## Installation

### Prerequisites

- Kubernetes cluster 1.16+
- kubectl configured to communicate with your cluster
- Helm 3.0+ (optional)

### Using Manifests

```bash
# Install CRDs
kubectl apply -f https://raw.githubusercontent.com/truskul/azdevops-agent-operator/main/config/crd/bases/devops.example.com_azuredevopsagentpools.yaml

# Install operator
kubectl apply -f https://raw.githubusercontent.com/truskul/azdevops-agent-operator/main/config/deploy/operator.yaml
```

### Using Helm

```bash
helm repo add azdevops-agent-operator https://truskul.github.io/azdevops-agent-operator/charts
helm repo update
helm install azdevops-agent-operator azdevops-agent-operator/azdevops-agent-operator
```

## Usage

### Create an Azure DevOps PAT token

1. In Azure DevOps, go to User Settings > Personal Access Tokens
2. Create a new token with the following permissions:
   - Agent Pools: Read & Manage
   - Deployment Groups: Read & Manage

### Create a Secret with the PAT token

```bash
kubectl create secret generic azdevops-pat --from-literal=token=YOUR_PAT_TOKEN
```

### Create an Agent Pool

Create a YAML file named `agentpool.yaml`:

```yaml
apiVersion: devops.example.com/v1alpha1
kind: AzureDevOpsAgentPool
metadata:
  name: build-agents
spec:
  organization: "your-organization"
  project: "your-project"
  poolName: "Linux-Build"
  
  agentNamePrefix: "k8s-build"
  minReplicas: 2
  maxReplicas: 10
  
  autoScaling:
    enabled: true
    targetQueueLength: 3
    scaleDownDelayMinutes: 10
  
  podTemplate:
    image: "your-agent-image:latest"
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
      limits:
        cpu: "2"
        memory: "4Gi"
    env:
      - name: EXTRA_CAPABILITY
        value: "docker"
    
  tokenSecret:
    name: "azdevops-pat"
    key: "token"
```

Apply the configuration:

```bash
kubectl apply -f agentpool.yaml
```

### Check the status

```bash
kubectl get azuredevopsagentpool
NAME          REPLICAS   BUSY   QUEUE   AGE
build-agents  3          1      0       10m
```

### View detailed status

```bash
kubectl describe azuredevopsagentpool build-agents
```

## Configuration Options

### AzureDevOpsAgentPool CRD

| Field                         | Description                                         | Default Value |
|-------------------------------|-----------------------------------------------------|---------------|
| `spec.organization`           | Azure DevOps organization name                      | -             |
| `spec.project`                | Azure DevOps project name                           | -             |
| `spec.poolName`               | Name of the agent pool                              | -             |
| `spec.agentNamePrefix`        | Prefix for agent names                              | -             |
| `spec.minReplicas`            | Minimum number of agent replicas                    | 1             |
| `spec.maxReplicas`            | Maximum number of agent replicas                    | 10            |
| `spec.autoScaling.enabled`    | Enable auto-scaling                                 | false         |
| `spec.autoScaling.targetQueueLength` | Target number of jobs in queue per agent     | 1             |
| `spec.autoScaling.scaleDownDelayMinutes` | Delay before scaling down                | 10            |
| `spec.podTemplate.image`      | Agent container image                               | -             |
| `spec.podTemplate.resources`  | Resource requests and limits                        | -             |
| `spec.podTemplate.env`        | Environment variables for agent container           | -             |
| `spec.tokenSecret.name`       | Name of the secret containing the PAT token         | -             |
| `spec.tokenSecret.key`        | Key in the secret containing the PAT token          | -             |

## Monitoring

The operator exposes metrics in Prometheus format at `/metrics` on port 8080. The following metrics are available:

- `azdevops_agent_pool_desired_replicas`: The desired number of agent replicas
- `azdevops_agent_pool_current_replicas`: The current number of agent replicas
- `azdevops_agent_pool_busy_agents`: The number of busy agents
- `azdevops_agent_pool_queue_length`: The number of jobs in queue
- `azdevops_agent_pool_reconcile_total`: Total number of reconciliation operations
- `azdevops_agent_pool_reconcile_errors_total`: Total number of reconciliation errors

## Building from Source

### Prerequisites

- Go 1.17+
- Kubebuilder
- Docker

### Build Steps

1. Clone the repository:
   ```bash
   git clone https://github.com/truskul/azdevops-agent-operator.git
   cd azdevops-agent-operator
   ```

2. Build and test:
   ```bash
   make test
   ```

3. Build the Docker image:
   ```bash
   make docker-build docker-push IMG=truskul/azdevops-agent-operator:latest
   ```

4. Deploy the operator:
   ```bash
   make deploy IMG=truskul/azdevops-agent-operator:latest
   ```

## Contributing

Contributions are welcome! Please check out our [contribution guidelines](CONTRIBUTING.md) for details.

### Development Environment

To set up a development environment:

```bash
# Install CRDs in your cluster
make install

# Run the controller locally
make run
```

## Roadmap

- [ ] Add support for self-signed certificates
- [ ] Implement advanced scaling algorithms
- [ ] Add support for agent capabilities and demands
- [ ] Integrate with Azure DevOps deployment pipelines
- [ ] Support for agent pool maintenance operations

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- Inspired by the [Jenkins Kubernetes Operator](https://github.com/jenkinsci/kubernetes-operator)
- Built with [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- Thanks to the [Kubernetes Operator community](https://github.com/operator-framework)