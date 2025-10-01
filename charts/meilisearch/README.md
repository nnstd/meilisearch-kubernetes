# meilisearch

A Helm chart for the Meilisearch search engine

![Version: 0.2.1](https://img.shields.io/badge/Version-0.2.1-informational?style=flat-square) ![AppVersion: v1.2.0](https://img.shields.io/badge/AppVersion-v1.2.0-informational?style=flat-square)

Helm works as a package manager to run pre-configured Kubernetes resources.

Meilisearch provides a customizable Helm chart, ready to deploy a [Meilisearch](https://github.com/meilisearch/meilisearch) instance on your Kubernetes cluster.

# Getting started

First of all, you will need a Kubernetes cluster up and running. If you are not familiar with how Kuberentes works or need some help with this step, please check the [Kubernetes documentation](https://kubernetes.io/docs/home/).

## Install kubectl

`kubectl` is the most commonly used CLI to handle a Kubernetes cluster. The installation instructions are [available here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

## Install helm

Helm CLI is a Command Line Interface which will automate chart management and installation on your Kubernetes cluster. To install Helm, follow the [Helm installation instructions](https://helm.sh/docs/intro/install/)

### Install Meilisearch chart

Clone this repository and install the chart

```bash
git clone https://github.com/meilisearch/meilisearch-kubernetes.git
cd meilisearch-kubernetes
# Replace <your-instance-name> with the name you would like to give to your service
helm install <your-service-name> charts/meilisearch
```

This command deploys Meilisearch on your Kubernetes cluster using the default configuration. The [Parameters](#parameters) section lists the parameters that can be configured during installation.

## Uninstalling the Chart

To uninstall/delete the `Meilisearch` deployment:

```bash
# Replace <your-instance-name> with the name of your deployed service
helm uninstall <your-service-name>
```

## Environment

The `environment` block allows to specify all the environment variables declared on [Meilisearch Configuration](https://www.meilisearch.com/docs/learn/configuration/instance_options#command-line-options-and-flags)

For production deployment, the `environment.MEILI_MASTER_KEY` is required. If `MEILI_ENV` is set to "production" without setting `environment.MEILI_MASTER_KEY`, then this chart will automatically create a secure `environment.MEILI_MASTER_KEY` as a secret.

You can also use `auth.existingMasterKeySecret` to use an existing secret that has the key `MEILI_MASTER_KEY`

## Horizontal Scaling with Sharding

This chart supports Meilisearch's sharding feature for horizontal scaling across multiple replicas. Sharding is available exclusively in **Meilisearch Enterprise Edition v1.19+**.

### Prerequisites

1. **Meilisearch Enterprise Edition**: Version 1.19 or higher
2. **License**: Required for production use (free for non-production environments under Business Source License 1.1)
3. **Master Key**: All instances must share the same `MEILI_MASTER_KEY`
4. **Primary Key**: Indexes must have a `primaryKey` configured before adding documents

### Enabling Sharding

To enable sharding in your Helm deployment:

```yaml
sharding:
  enabled: true
  replicaCount: 3  # Number of nodes in the sharded cluster
  network:
    experimentalFeatures: true
    auth:
      # Option 1: Let Helm generate random API keys (development)
      searchApiKey: ""
      writeApiKey: ""
      
      # Option 2: Use existing secret (recommended for production)
      existingSecret: "my-sharding-keys-secret"
      # The secret should contain 'searchApiKey' and 'writeApiKey' keys by default
      
      # If your existing secret uses different key names:
      existingSecretSearchApiKey: "customSearchKeyName"
      existingSecretWriteApiKey: "customWriteKeyName"

persistence:
  enabled: true
  storageClass: "standard"
  size: 10Gi
```

### How It Works

When sharding is enabled:

1. A **headless service** is created for StatefulSet pod-to-pod communication
2. Each pod gets a unique **persistent volume** for its data partition
3. A **network configurator sidecar** automatically configures the network topology
4. Documents sent to any node are automatically distributed across all shards
5. Each node indexes only its assigned subset of documents using consistent hashing

### Searching Sharded Indexes

To search across a sharded index, use [Meilisearch federated search](https://www.meilisearch.com/docs/learn/multi_search/performing_federated_search):

```bash
curl -X POST 'http://meilisearch:7700/multi-search' \
  -H 'Authorization: Bearer YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "federation": {},
    "queries": [
      {
        "indexUid": "movies",
        "q": "star wars",
        "federationOptions": { "remote": "node-0" }
      },
      {
        "indexUid": "movies",
        "q": "star wars",
        "federationOptions": { "remote": "node-1" }
      },
      {
        "indexUid": "movies",
        "q": "star wars",
        "federationOptions": { "remote": "node-2" }
      }
    ]
  }'
```

### Architecture

```
┌─────────────────────────────────────────┐
│         Load Balancer / Ingress         │
└────────────────┬────────────────────────┘
                 │
      ┌──────────┼──────────┐
      ▼          ▼          ▼
  ┌──────┐  ┌──────┐  ┌──────┐
  │Node-0│  │Node-1│  │Node-2│
  └───┬──┘  └───┬──┘  └───┬──┘
      │         │         │
  ┌───▼──┐  ┌───▼──┐  ┌───▼──┐
  │ PVC  │  │ PVC  │  │ PVC  │
  │ 10Gi │  │ 10Gi │  │ 10Gi │
  └──────┘  └──────┘  └──────┘
```

### Important Notes

- **Persistence**: When sharding is enabled with persistence, each pod gets its own PVC (using volumeClaimTemplates)
- **Scaling**: Changing the number of replicas after initial deployment requires careful data rebalancing
- **Network Configuration**: Automatically handled by the sidecar container
- **API Keys**: The `searchApiKey` and `writeApiKey` are used for secure inter-node communication
- **Experimental**: Sharding is an experimental feature as of Meilisearch v1.19

For more information, see the [official Meilisearch sharding documentation](https://www.meilisearch.com/blog/horizontal-scaling-with-sharding).

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| affinity | object | `{}` | Affinity for pod assignment |
| auth.existingMasterKeySecret | string | `""` | Use an existing Kubernetes secret for the MEILI_MASTER_KEY |
| command | list | `[]` | Pod command |
| container.containerPort | int | `7700` |  |
| containers | list | `[]` | Additional containers for pod |
| customLabels | object | `{}` | Additional labels to add to all resources |
| envFrom | list | `[]` | Additional environment variables from ConfigMap or secrets |
| environment.MEILI_ENV | string | `"development"` | Sets the environment. Either **production** or **development** |
| environment.MEILI_NO_ANALYTICS | bool | `true` | Deactivates analytics |
| fullnameOverride | string | `""` | String to fully override meilisearch.fullname |
| image.pullPolicy | string | `"IfNotPresent"` | Meilisearch image pull policy |
| image.pullSecret | string | `nil` | Secret to authenticate against the docker registry |
| image.repository | string | `"getmeili/meilisearch"` | Meilisearch image name |
| image.tag | string | `"v1.2.0"` | Meilisearch image tag |
| image.digest | string | `""` | 	Meilisearch image digest in the way sha256:aa.... Please note this parameter, if set, will override the tag |
| ingress.annotations | object | `{}` | Ingress annotations |
| ingress.className | string | `"nginx"` | Ingress ingressClassName |
| ingress.enabled | bool | `false` | Enable ingress controller resource |
| ingress.hosts | list | `["meilisearch-example.local"]` | List of hostnames |
| ingress.path | string | `"/"` | Path within the host |
| ingress.tls | list | `[]` | TLS specification |
| livenessProbe.InitialDelaySeconds | int | `0` |  |
| livenessProbe.periodSeconds | int | `10` |  |
| livenessProbe.timeoutSeconds | int | `10` |  |
| nameOverride | string | `""` | String to partially override meilisearch.fullname |
| nodeSelector | object | `{}` | Node labels for pod assignment |
| persistence.accessMode | string | `"ReadWriteOnce"` | PVC Access Mode |
| persistence.annotations | object | `{}` | Additional annotations for PVC |
| persistence.enabled | bool | `false` | Enable persistence using PVC |
| persistence.existingClaim | string | `""` | Existing PVC |
| persistence.size | string | `"10Gi"` | PVC Storage Request |
| persistence.storageClass | string | `"-"` | PVC Storage Class |
| persistence.volume.mountPath | string | `"/meili_data"` |  |
| persistence.volume.name | string | `"data"` |  |
| podAnnotations | object | `{}` |  |
| podLabels | object | `{}` | Additional labels to add to the pod(s) only |
| podSecurityContext.fsGroup | int | `1000` |  |
| podSecurityContext.fsGroupChangePolicy | string | `"OnRootMismatch"` |  |
| podSecurityContext.runAsGroup | int | `1000` |  |
| podSecurityContext.runAsNonRoot | bool | `true` |  |
| podSecurityContext.runAsUser | int | `1000` |  |
| readinessProbe.InitialDelaySeconds | int | `0` |  |
| readinessProbe.periodSeconds | int | `10` |  |
| readinessProbe.timeoutSeconds | int | `10` |  |
| replicaCount | int | `1` | Number of Meilisearch pods to run |
| resources | object | `{}` | Resources allocation (Requests and Limits) |
| securityContext.allowPrivilegeEscalation | bool | `false` |  |
| securityContext.capabilities.drop[0] | string | `"ALL"` |  |
| securityContext.readOnlyRootFilesystem | bool | `true` |  |
| service | object | `{"annotations":{},"port":7700,"type":"ClusterIP"}` | Service HTTP port |
| service.annotations | object | `{}` | Additional annotations for service |
| service.type | string | `"ClusterIP"` | Kubernetes Service type |
| serviceAccount.annotations | object | `{}` | Additional annotations for created service account |
| serviceAccount.create | bool | `true` | Should this chart create a service account |
| serviceAccount.name | string | `""` | Custom service account name, if not created by this chart |
| serviceMonitor | object | `{"additionalLabels":{},"enabled":false,"interval":"1m","metricRelabelings":[],"relabelings":[],"scrapeTimeout":"10s","targetLabels":[],"telemetryPath":"/metrics"}` | Monitoring with Prometheus Operator |
| serviceMonitor.additionalLabels | object | `{}` | Set of labels to transfer from the Kubernetes Service onto the target |
| serviceMonitor.enabled | bool | `false` | Enable ServiceMonitor to configure scraping |
| serviceMonitor.interval | string | `"1m"` | Set scraping frequency |
| serviceMonitor.metricRelabelings | list | `[]` | MetricRelabelConfigs to apply to samples before ingestion |
| serviceMonitor.relabelings | list | `[]` | Set relabel_configs as per https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config |
| serviceMonitor.scrapeTimeout | string | `"10s"` | Set scraping timeout  |
| serviceMonitor.targetLabels | list | `[]` | Set of labels to transfer from the Kubernetes Service onto the target |
| serviceMonitor.telemetryPath | string | `"/metrics"` | Set path to metrics |
| startupProbe.InitialDelaySeconds | int | `1` |  |
| startupProbe.failureThreshold | int | `60` |  |
| startupProbe.periodSeconds | int | `1` |  |
| startupProbe.timeoutSeconds | int | `1` |  |
| tolerations | list | `[]` | Tolerations for pod assignment |
| volumeMounts | list | `[]` | Additional volumes to mount on pod |
| volumes | list | `[]` | Additional volumes for pod |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.11.0](https://github.com/norwoodj/helm-docs/releases/v1.11.0)
