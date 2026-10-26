# Cedana Helm chart

Cedana is automated GPU checkpointing and migration infrastructure that increases the useful work your GPUs deliver.

This chart installs Cedana on Kubernetes: a controller, and a helper DaemonSet that runs the Cedana daemon on each node. A workload opts in with one environment variable.

## Install

The chart needs three values from Cedana: your org URL, an auth token and a cluster ID. To get the cluster ID, register the cluster on the Clusters page of the Cedana platform ([installation guide](https://docs.cedana.ai/cedana-kubernetes/installation)). You need a Cedana account for this. Reach out to us through your contact at Cedana or founders@cedana.ai to get an account!

The install command that Artifact Hub shows has none of these values, so use this one:

```sh
export CEDANA_URL=<your-org-url>
export CEDANA_AUTH_TOKEN=<your-auth-token>
export CEDANA_CLUSTER_ID=<cluster-id>

helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN
```

On clusters with CPU nodes only, add `--set=config.pluginsGpuVersion=none`:

```sh
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN \
  --set=config.pluginsGpuVersion=none
```

## What the chart installs

| Object | Kind | Where it runs | Access it needs |
|---|---|---|---|
| `cedana-manager` (the controller) | Deployment | release namespace | Cluster role: read and update nodes and namespaces; create, update and delete pods; read deployments, daemonsets and statefulsets; create and delete jobs; create persistent volume claims; update persistent volumes; read, create and update config maps; create, update and delete mutating webhook configurations. In the release namespace: create and update secrets; create, update and delete services. Full list in `templates/manager-rbac.yaml` |
| `cedana-helper` | DaemonSet | every node, or the nodes that match `daemonHelper.nodeSelector` | Privileged, host PID, IPC and network namespaces, host paths. Installs the Cedana daemon and its dependencies on the host |
| `cedana-health-check` | DaemonSet | the same nodes | Privileged, host PID, IPC and network namespaces |
| `cedana` | RuntimeClass | cluster | GPU pods select it with `runtimeClassName: cedana` |
| `cedana.ai/not-ready` | Node taint | each Cedana node while Cedana starts | The controller adds it and removes it when Cedana is ready. Turn off with `controllerManager.taintNodes=false` |
| `cedana-csx` | DaemonSet, optional | when `config.checkpointDir` is `csx://`: every node, or the nodes that match `csx.nodeSelector` (not `daemonHelper.nodeSelector`). Needs `csx.nfsPath` | Privileged, host PID, IPC and network namespaces |
| `cedana-dynamo-watcher` | Deployment, optional | release namespace, when `dynamoWatcher.enabled` is true | Cluster role |
| `cedana-pd-disagg` webhook, with the `cedana-pd-webhook` Service and its serving certificate Secret | MutatingWebhookConfiguration, Service and Secret, optional, created by the controller | cluster for the webhook configuration, release namespace for the Service and Secret, when `controllerManager.pdDisaggWebhook.enabled` is true | Changes only pods annotated `cedana.ai/pd-disagg` |
| `kubernetes-event-exporter` | Deployment, optional | release namespace, when `clusterMetrics.enabled` and `clusterMetrics.vector.enabled` are true | Cluster role: read events, pods, namespaces and jobs |
| Prometheus, Vector, dcgm-exporter | Subcharts, optional | release namespace, when `clusterMetrics.enabled` and each subchart flag are true | Their own charts |
| Config maps, a secret with the auth token, services | Configuration | release namespace | None |

## What changes in your pod spec

Set `CEDANA_CHECKPOINT` to a unique ID. Cedana keys the latest checkpoint of the workload to that ID. After a checkpoint, apply the same spec again, and the next start restores from the checkpoint instead of a cold start. GPU pods also set `runtimeClassName: cedana`.

```yaml
spec:
  runtimeClassName: cedana
  containers:
    - name: my-workload
      env:
        - name: CEDANA_CHECKPOINT
          value: my-workload-1
```

The docs explain [automatic checkpoint and restore](https://docs.cedana.ai/cedana-kubernetes/automation) and [manual checkpoint and restore](https://docs.cedana.ai/cedana-kubernetes/cr).

## Prerequisites

- A Cedana account and a registered cluster (see Install).
- Nodes that allow privileged pods, and root access on the nodes (see the table above).
- containerd on each node. The helper adds a `cedana` runtime to the containerd configuration, and it also handles the containerd that k3s and RKE2 run.
- For GPU workloads: NVIDIA GPUs. The Cedana GPU plugin supports NVIDIA driver versions from 452 to 570, on amd64 and arm64 nodes. The [GPU support table](https://docs.cedana.ai/daemon/checkpoint-restore/gpu) in the docs has the details.
- Cedana's CI tests the chart on K3s, EKS and GKE on each push to `main`.

## Job sizes and limits

Cedana works with single-GPU, multi-GPU on a single node and multi-node GPUs.

A checkpoint records the GPU, driver, engine and model versions it was taken against. If any of them changes, the checkpoint is invalid and the workload cold-starts instead. The full list is in [What cannot be checkpointed in a GPU workload?](https://cedana.com/blog/gpu-checkpointing-limitations/)

## Uninstall

```sh
helm uninstall cedana -n cedana-system
```

Before Helm removes the chart, a pre-delete hook starts a `cedana-uninstaller` DaemonSet on each node. It runs `cedana k8s destroy` on the host (and the CSX uninstall script when `config.checkpointDir` is `csx://`), removes the `cedana.ai/not-ready` taint from all nodes, removes the `cedana-pd-disagg` webhook configuration, and then deletes the release namespace. Install the chart in its own namespace, because uninstall deletes that namespace.

`cedana k8s destroy` stops and removes the `cedana` service. It deletes Cedana's configuration, logs, binaries and plugins (the `criu` binary too), the kubelet drop-in `99-cedana.conf` and the `cedana` runtime in the containerd configuration. If it changed the containerd configuration, it restarts containerd (or k3s or RKE2). The kubelet drop-in stops applying at the next kubelet restart.

These stay on each host:

- The packages the helper installed with apt or yum, such as the CRIU libraries.
- The `/dev/shm` line in `/etc/fstab`, if `hostConfig.shmConfig.enabled` was true. Remove it by hand.
- On containerd 2.x, the `conf.d` import that Cedana added to the containerd configuration. Other files in `conf.d` can depend on it.
- The NVIDIA library path in `/etc/ld.so.conf.d/nvidia.conf`, if the helper added it.

## Configuration

`config.url`, `config.authToken` and `config.clusterId` are required values. When `config.preExistingSecret` names a secret you created, `config.authToken` is not required, but the pods then also read a ConfigMap with the secret's name: create it too, with the same keys as the chart's `cedana-config` ConfigMap. The full reference is in the [configuration docs](https://docs.cedana.ai/cedana-kubernetes/configuration) and in `values.yaml`.

### CRIU device plugins

The CRIU `nvidia-dev` and `infiniband` plugins are installed on every node by default.
They let CRIU checkpoint processes holding open `/dev/nvidia*` (e.g. NVML users) or
`/dev/infiniband/*` fds, and are inert on nodes without those devices. They follow
`config.pluginsCriuVersion`. To opt out (the plugin is removed from the node):

```bash
--set config.pluginsCriuNvidiaDev=false
--set config.pluginsCriuInfiniband=false
```

### Restricting Cedana to specific nodes

By default the Cedana helper runs on every node in the cluster. To limit it to a
specific set of nodes, label those nodes and set a matching `nodeSelector`:

- Label the nodes you want Cedana to run on:
```bash
kubectl label nodes <node-name> cedana.ai/enabled=true
```

- Set the selector at install time (or in a values override):
```bash
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN \
  --set-string daemonHelper.nodeSelector."cedana\.ai/enabled"=true
```

The Cedana controller reads this same `nodeSelector` to decide which nodes to
manage, so it only applies the `cedana.ai/not-ready` taint to in-scope nodes.
Nodes outside the selector are never tainted, and if you narrow the selector
later, any existing `not-ready` taint is removed from the now-out-of-scope
nodes automatically. You do **not** need to configure the taint or its
toleration yourself. Leave the built-in `cedana.ai/not-ready` toleration in
place; it is required for Cedana to function.

> This is the common case ("run Cedana on these nodes"). It does **not**
> prevent other workloads from also running on those nodes. To *dedicate*
> nodes exclusively to Cedana, see [Dedicated node for the controller](#dedicated-node-for-the-controller)
> below, which additionally taints the node to repel other workloads.

### Managing node taints yourself

By default the controller manages a `cedana.ai/not-ready:NoSchedule` taint on each
node: it taints the node while Cedana is starting up there and removes the taint
once Cedana is ready, gating other workloads off the node until it can
checkpoint/restore. If you'd rather manage node readiness yourself (via your own
tooling, GitOps, or an admission controller), disable it:

```bash
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN \
  --set controllerManager.taintNodes=false
```

When disabled, the controller is completely hands-off. It will neither add nor
remove the `cedana.ai/not-ready` taint, and you are responsible for keeping
workloads off nodes until Cedana is ready. This is enabled (`true`) by default.

### Restores for disaggregated prefill and decode workers

To checkpoint and restore Dynamo/vLLM or SGLang prefill and decode workers, turn on the controller's P/D disagg webhook:

```bash
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN \
  --set controllerManager.pdDisaggWebhook.enabled=true
```

The controller then creates the webhook's `cedana-pd-webhook` Service, its self-signed serving cert (Secret `cedana-pd-webhook-pd-webhook-tls`) and the `cedana-pd-disagg` MutatingWebhookConfiguration itself; there is nothing else to install. It sets up only pods annotated `cedana.ai/pd-disagg` (see [Deploying with Dynamo](https://docs.cedana.ai/cedana-inference/deploying-with-dynamo)). Setting it back to `false` removes the webhook configuration and Service on the controller's next start. The RBAC it needs is part of the chart either way. This is disabled (`false`) by default.

### Dedicated node for the controller

To run the Cedana Controller exclusively on a specific node:

- Taint the node to prevent general workloads:
```bash
kubectl taint node <node-name> dedicated=cedana-manager:NoSchedule
```

- Label the node to target it for scheduling:
```bash
kubectl label node <node-name> dedicated=cedana-manager
```

- Put the toleration and the node affinity in a values file, for example `dedicated.yaml`. Keep the `cedana.ai/not-ready` toleration, which Cedana needs:
```yaml
controllerManager:
  tolerations:
    - key: "cedana.ai/not-ready"
      operator: "Exists"
      effect: "NoSchedule"
    - key: "dedicated"
      operator: "Equal"
      value: "cedana-manager"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "dedicated"
                operator: "In"
                values:
                  - "cedana-manager"
```

- Install with that file:
```bash
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm \
  --create-namespace -n cedana-system \
  --set=config.clusterId=$CEDANA_CLUSTER_ID \
  --set=config.url=$CEDANA_URL \
  --set=config.authToken=$CEDANA_AUTH_TOKEN \
  -f dedicated.yaml
```

### Shared memory (SHM)

For workloads that require large shared memory, you can optionally increase the `/dev/shm` size on all nodes:

```bash
# Enable SHM configuration with default 10G size
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm --create-namespace -n cedana-system \
--set config.url=$CEDANA_URL \
--set config.authToken=$CEDANA_AUTH_TOKEN \
--set config.clusterId=$CEDANA_CLUSTER_ID \
--set hostConfig.shmConfig.enabled=true

# Customize SHM size (e.g., 20G)
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm --create-namespace -n cedana-system \
--set config.url=$CEDANA_URL \
--set config.authToken=$CEDANA_AUTH_TOKEN \
--set config.clusterId=$CEDANA_CLUSTER_ID \
--set hostConfig.shmConfig.enabled=true \
--set hostConfig.shmConfig.size="20G"
```

### EKS Pod Identity for S3 checkpoints

The helper and health-check DaemonSets use a dedicated `cedana-helper` ServiceAccount. The chart does not install the EKS Pod Identity Agent, create IAM roles or policies, or create Pod Identity associations.

1. Install the EKS Pod Identity Agent add-on on the cluster.
2. Create an IAM role with a least-privilege policy for the checkpoint bucket. Typical permissions are `s3:ListBucket` on the bucket and `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:AbortMultipartUpload`, and multipart-list permissions on the checkpoint prefix.
3. Associate that role with the chart namespace and the `cedana-helper` ServiceAccount.
4. Install or upgrade with:

```yaml
config:
  checkpointDir: s3://my-checkpoint-bucket/cedana
  awsCredentialsMode: eksPodIdentity
  awsRegion: us-east-1

daemonHelper:
  serviceAccount:
    create: true
    name: cedana-helper
    automount: false
    annotations: {}
```

To use an externally managed ServiceAccount, set `daemonHelper.serviceAccount.create: false` and provide its non-empty `name`. Use `config.awsCredentialsMode: ambient` for IRSA, node roles, or other AWS default-chain credential sources. Static mode remains the default and continues to read `config.awsAccessKeyId` and `config.awsSecretAccessKey`.

### Monitoring

When cluster metrics are enabled, you must populate the metrics S3 bucket name explicitly. The S3 region and credentials will automatically fall back to the main `config` settings if omitted.

The monitoring stack includes **Prometheus** (for metrics collection) and **Vector** (for shipping logs/metrics to S3). Individual subcharts can be enabled via `clusterMetrics.prometheus.enabled` and `clusterMetrics.vector.enabled` respectively.

Note that `clusterMetrics.enabled` must be set to `true` to use these subcharts. If `clusterMetrics.enabled` is `false` while any sub-component is enabled, the installation will fail validation.

The published chart already holds the Prometheus and Vector subcharts. Install with metrics enabled (e.g. enabling both Prometheus and Vector):
```bash
helm upgrade -i cedana oci://registry-1.docker.io/cedana/cedana-helm --create-namespace -n cedana-system \
  --set config.url=$CEDANA_URL \
  --set config.authToken=$CEDANA_AUTH_TOKEN \
  --set config.clusterId=$CEDANA_CLUSTER_ID \
  --set config.awsRegion=$AWS_REGION \
  --set config.awsAccessKeyId=$AWS_ACCESS_KEY_ID \
  --set config.awsSecretAccessKey=$AWS_SECRET_ACCESS_KEY \
  --set clusterMetrics.enabled=true \
  --set clusterMetrics.prometheus.enabled=true \
  --set clusterMetrics.vector.enabled=true \
  --set clusterMetrics.s3.bucket=$METRICS_S3_BUCKET
```

If your cluster has NVIDIA GPUs and you want to monitor GPU metrics (utilization, memory, temperature, etc.), set `clusterMetrics.gpu.enabled=true` as well. Note that by default, the GPU exporter DaemonSet will only schedule pods on GPU-enabled nodes labeled with `nvidia.com/gpu.present=true`:

```bash
  --set clusterMetrics.gpu.enabled=true
```

## Security

If you want to use our tools, but have specific security requirements reach out and let us know
about it.

Report a security issue to contact@cedana.ai.

- Daemonset requires privilege escalation for daemon container, and it updates the host directly with
  required dependencies and our cedana-daemon application.
- Currently our controller requires access to all pods to be able to list, checkpoint and restore them.
- SHM configuration requires privileged access to modify host filesystem and mount points.

## More

- [Cedana Kubernetes docs](https://docs.cedana.ai/cedana-kubernetes/installation)
- [What changes on your cluster when you install Cedana](https://cedana.com/blog/cedana-installation-cluster-changes/)
- [How Cedana checkpoint and restore works](https://cedana.com/blog/how-cedana-checkpoint-restore-works/)
- [Sample workloads](https://github.com/cedana/cedana-samples)
- [Report an issue](https://github.com/cedana/cedana-helm-charts/issues)
- [cedana.com](https://cedana.com)
