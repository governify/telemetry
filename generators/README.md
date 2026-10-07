# Generators

How to connect things to the gateway (`telemetry.governify.io:4317`; plain and unauthenticated today).

| To measure | Do |
|---|---|
| **A server** (whatever it runs) | [`docker-host/`](docker-host/) on it, installing Docker first if it has none |
| **An app in Docker** | Nothing: `docker-host` already measures every container |
| **An app in Kubernetes** | [One-time setup](#apps-in-kubernetes-one-time-setup) per cluster, then nothing per app |

## Add a server

On the server, with Docker and docker-compose installed and this repo's `generators/docker-host/` folder on it:

```bash
cd generators/docker-host
cp .env.example .env && nano .env     # HOST_NAME=<unique server name>
docker-compose up -d && docker logs otel-agent     # no errors
```

Done when the server appears in the NOC servers table (1-2 min). `HOST_NAME` is the only required value: set it by hand,
don't rely on the hostname. Optional in `.env`: `GATEWAY_OTLP_ENDPOINT`, `DEPLOYMENT_ENVIRONMENT` (default `production`).

The agent runs as root (reads `/` and the Docker socket): never publish its ports publicly. After editing
`otel-agent-config.yaml`: `docker-compose up -d --force-recreate otel-agent`.

## Apps in Docker

Nothing to install: every running container shows up as a service (app = compose project, service = compose service) with
status, CPU, memory and restarts. To choose the names yourself, label the container:

```yaml
labels:
  telemetry.service.namespace: my-app
  telemetry.service.name: my-api
```

Stopped containers disappear (no "exited" state). Req/s, 5xx and p95 show "–" for apps that don't send HTTP metrics themselves.

## Apps in Kubernetes: one-time setup

Once per cluster (the server itself is still measured by `docker-host`). On a machine with `kubectl` and `helm` pointing at
the cluster: edit `DEPLOYMENT_ENVIRONMENT` and `K8S_CLUSTER_NAME` in both values files, then:

```bash
cd generators/kubernetes
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm upgrade --install otel-agent   open-telemetry/opentelemetry-collector --version 0.173.1 -n telemetry --create-namespace -f values-agent.yaml
helm upgrade --install otel-cluster open-telemetry/opentelemetry-collector --version 0.173.1 -n telemetry -f values-cluster.yaml
```

Each namespace becomes an app, with its pods, deployments, restarts and events.

- `values-agent.yaml` (DaemonSet): pod / container metrics from the kubelet. `values-cluster.yaml` (1 replica): deployments, pod phases, events.
- Tip: use the Kubernetes node name (`kubectl get nodes`) as the server's `HOST_NAME`, so the pods can be tied to that server.
- Several clusters on one server (k3s + kind...): different `K8S_CLUSTER_NAME`, `--kube-context` on every `helm` call, and avoid
  identical namespace names (`default`, `kube-system`): they would merge into one app.
- Self-signed kubelet cert (k3s, minikube, on-prem): uncomment `insecure_skip_verify` in `values-agent.yaml`.
- **Templates, not yet run on a real cluster.** Untested: RBAC, kubelet auth, `k8s_attributes`, `k8s_cluster` metric names.
  Pod logs are not collected.

## Label contract

Dashboards and recording rules rely on these resource attributes (Prometheus turns `.` into `_`). The promoted list is
`otlp.promote_resource_attributes` in `gateway-and-backends/config/prometheus.yml`: keep both in sync.

| Attribute | Label | Meaning |
|---|---|---|
| `service.namespace` | `service_namespace` | **The application.** Without it nothing shows up (on k8s the gateway falls back to the k8s namespace) |
| `service.name` | `service_name` | A service of the app |
| `service.instance.id` | `service_instance_id` | One replica (unique per pod / container / process) |
| `deployment.environment.name` | `deployment_environment_name` | Environment |
| `service.version` | `service_version` | Version |
| `host.name` | `host_name` | **The server.** The agents force it to `HOST_NAME` |
| `k8s.*`, `container.*` | `k8s_*`, `container_*` | Pod / node / workload / container drill-downs |

A service is "up" while its metrics arrive (< 3 min) and "missing" for 24 h after they stop.
