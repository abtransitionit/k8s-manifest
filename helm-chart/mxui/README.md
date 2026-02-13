
[←]: ../README.md
[`01-dep.yaml`]: ./templates/01-dep.yaml
[`02-svc.yaml`]: ./templates/02-svc.yaml

# [←] MxUI
## Definition
- a contenerized application
- exposes several UIs (**U**ser **I**nterface**s**) like firefox
- example of image used:
  - `jlesage/firefox:latest`

## Requirement
| Property              | Value                   | Notes                          |
| --------------------- | ----------------------- | ------------------------------ |
| **Application Name**  | `mxui`                  | |
| **Storage Type**      | Ephemeral (Pod storage) | Data is stored in **Ephemeral** container      |
| **Storage Persistence** | None                    | **Data is lost** on pod restart, reschedule       |

   |
# Manifest

list of the application's manifests

| id| Yaml| kind|Description|
|-|-|-|-|
| 01| [`01-dep.yaml`]| `deploy:mxui-deploy`|Deployment for firefox|
| 02| [`02-svc.yaml`]| `service:mxui-svc`|ensures a **simple** well known **Ip:port/Dns** for the browser service|



# dep.yaml

1. `kubectl apply -f cimreg-sc.yaml`
1. Kubernetes creates a **Deployment** named `cimreg-deploy` in the `cimreg` namespace.
1. The Deployment says: “I want **1 running pod** at all times.”
1. Kubernetes creates **one pod** matching the labels:
    - `app.kubernetes.io/name: cimreg`
    - `app.kubernetes.io/component: registry`
1. To start the pod, Kubernetes sees:
    - The pod needs a **PVC** named `cimreg-pvc`
    - The PVC uses a **SC** (from earlier)
1. Because of `WaitForFirstConsumer`:
    - Kubernetes first chooses **one specific node** for the pod
    - The pod is **bound to that node**
1. On that chosen node:
    - The `SC` provisioner creates:
      * one **PV**
      * one **directory on the host**, under
    `/var/lib/cimreg-local/…`
    - This directory becomes the **real storage** for the registry
1. The pod starts on that node:
    - A container `cregistry` is launched
    - Image: `registry:3.0.0`
    - The container listens on port **5000**
1. Inside the container:
    - The host directory (via PV → PVC)
    - is mounted into the container at `/var/lib/registry`
1. When the registry writes images:
    - Data goes to `/var/lib/registry` inside the container
    - Which is actually stored on the **node filesystem**
1. If the pod is deleted, crashes or restarts:
    - Kubernetes recreates the pod
    - The pod is scheduled on the same node because
      - the PV is local
      - the PVC is bound to that PV
      - Kubernetes must schedule the pod on that same node (desired state).
    - The same storage directory is reused
    - Registry data is still there
1. If the Deployment is deleted:
    - The pod is deleted
    - The PVC may be deleted
    - The PV and the data **remain on the node** (`Retain`)


**Result**:

👉 **This manifest ensures** that exactly one Docker registry pod runs on a single node and stores its data persistently on that node’s local filesystem via a PVC-backed volume.

**Comment**

when scheduling the node for the pod, the scheduler picks a node:
  - Normally, it could choose any node that fits.
  - But the pod uses a PVC bound to a local PV, which only exists on one node.
  - So the scheduler has only one valid choice → the same node.

# svc.yaml

1. `kubectl apply -f cimreg-svc.yaml`
1. Kubernetes creates a **Service** named `cimreg-svc` in the `cimreg` namespace.
1. The Service is of type **ClusterIP**:
    - It gets an **internal cluster IP**
    - It is **not reachable from outside the cluster**
1. The Service looks for pods with these labels:
    - `app.kubernetes.io/name: cimreg`
    - `app.kubernetes.io/component: registry`
1. Kubernetes finds the **registry pod** created by the Deployment.
1. The Service creates an internal **network mapping**:
    - Service IP : port **5000**
    - → forwards traffic to the pod
    - → to the container port **5000**
1. Inside the cluster:
    - Any pod can reach the registry via
  `cimreg-svc:5000`
    - Or via the Service IP on port 5000
1. If the registry pod restarts:
    - The Service stays the same
    - Traffic is automatically redirected to the new pod

**Result**:
- **Service gives the registry a stable internal address**
- **No ingress, no TLS, no auth, cluster-internal only** 🔒📦
- 👉 This manifest ensures 
  - the service provide by the container/pod is reachable by any pods in the cluster, even on **rescheduled**
  - the `IP:port/Dnsname` is **internal** and **well known**

# Debug
**restart the provisioner**
```sh
# clear cache, kill old pods and restart new one
kubectl rollout restart deployment kbe-openebs-localpv-provisioner -n openebs
```
**wait end of deplyment**
```shell
# wait until Pods are Running and Ready.
kubectl rollout status deploy/cimreg-deploy -n cimreg
```
**force create the sc hostPath**
```sh
# on each node
sudo mkdir -p /var/lib/cimreg-local
sudo chmod 755 /var/lib/cimreg-local
```


## Terminomogy
|Term|Meaning|
|-|-|
|UI|**U**ser **I**nterface|



# Test
## Test the ClusterIp
> ❗ `clusterIP` is the base for other kind of services (**LB**, **NodePort**)

**from inside the cluster**
- create an interactive shell
```bash
# define var
lPodName="mx-registry-client-test"
lImageName="docker:27-cli"
lNs="cimreg"

## create an get interactive shell
kubectl run $lPodName \
  --image=$lImageName \
  -n $lNs \
  --rm -it \
  --restart=Never \
  -- /bin/sh -l

```

**Inside the pod**:

```sh
wget -qO- http://registry.registry.svc.cluster.local:5000/v2/_catalog
```

Expected output:

```json
{"repositories":[]}
```


## Push images (node/containerd side)
**context**
- From a node or CI running `containerd` or `Docker` (ie. a `container runtime`)
- Registry endpoint:`registry.registry.svc.cluster.local:5000`

Example:

```bash
docker tag alpine registry.registry.svc.cluster.local:5000/alpine:latest
docker push registry.registry.svc.cluster.local:5000/alpine:latest
```

(If using containerd + insecure registry, you’ll need the usual `hosts.toml` config.)



## Todo

* 🔒 **No auth yet** (can add htpasswd later)
* 🔐 **No TLS yet** (fine for private/internal)
* 📦 **Single replica only** with hostpath
* 🧠 Data lives on **one node** → reschedule = data stays on that node


* add **auth (htpasswd)**
* add **TLS with cert-manager**
* pin the registry pod to a **specific node**
* or make it **CI-friendly** (GitLab / Kaniko)


## Todo
**folder layout**
```yaml
# use kustomization.yaml later
cimreg/
 ├── 00-namespace.yaml
 ├── 01-storage.yaml
 ├── 02-config.yaml
 ├── 03-deployment.yaml
 ├── 04-service.yaml
 ├── 05-ingress.yaml
 ├── 06-security.yaml
```


**Todo**
| Concern| v1|V2
| ----------------------| ---------------------|-|
| Persistence| PVC required|
| Authentication| -|htpasswd / token auth|
| TLS||mandatory in prod
| Exposure| ClusterIP|Ingress
| Storage backend| filesystem|
| Garbage collection||operational procedure
| High availability||probably later
| Image retention policy| optional|

**Test**
|||
|-|-|
|Namespace|`kubectl get ns cimreg`
|Storage|`kubectl describe pvc`<br>• `kubectl get events`
|Deployment|• `kubectl logs`<br>• `kubectl exec`<br>• `kubectl port-forward`

# Check
**folder exists acroos all node**
```sh
kubectl get nodes -o name | xargs -I{} kubectl debug {} -it --image=busybox -- ls -ld /var/lib/cimreg-local
```

# Question
- Does this registry need to be accessible by a specific User ID (UID), or is standard root access enough for your setup?

# Todo
|  ID | Area       | Item                                       | Current Status | Reason for Deferral               | Notes / When                                     |
| --: | ---------- | ------------------------------------------ | -------------- | --------------------------------- | ------------------------------------------------ |
| T01 | Config     | Inject `cimreg-cm` into `cimreg-ds`        | Deferred       | Path is stable for now            | Enables future path refactor without touching DS |
| T02 | Ops        | Reduce DaemonSet “noise” (exit after prep) | Deferred       | DS must react to node joins       | Acceptable idle pods                             |
| T03 | Governance | Revisit `kube-system` placement            | Deferred       | Node-wide infra concern           | Possibly move to `cimreg-system` later           |
| T04 | Runtime    | Add CPU/memory requests & limits           | Deferred       | Early functional phase            | Required before load / multi-tenant              |
| T05 | Health     | Add readiness & liveness probes            | Deferred       | Registry is single-user for now   | Mandatory before exposing service                |
| T06 | Scheduling | Add node affinity / pinning                | Deferred       | `WaitForFirstConsumer` sufficient | Needed if node topology becomes constrained      |
| T07 | Security   | Drop root privileges where possible        | Deferred       | HostPath prep requires root       | Registry container can be non-root later         |
| T08 | Networking | Add Service (ClusterIP)                    | Planned        | Needed for internal access        | Next logical step                                |
| T09 | Security   | Add authentication (htpasswd/token)        | Planned        | Registry currently open           | Required before multi-user                       |
| T10 | Security   | TLS termination                            | Planned        | No ingress yet                    | Mandatory for Docker clients                     |
| T11 | Policy     | Add NetworkPolicy (Cilium)                 | Planned        | Single workload for now           | Lock down access paths                           |
| T12 | Data       | Backup / restore strategy                  | Planned        | Single-node storage               | Snapshot or rsync-based                          |
| T13 | Ops        | Garbage collection policy                  | Planned        | Registry growth unmanaged         | Required before long-term use                    |
| T14 | Lifecycle  | Add PodDisruptionBudget                    | Planned        | Single replica                    | Needed before node maintenance                   |
| T15 | Observability | Disable OpenTelemetry tracing | Done   | Noise removed, tracing deferred |
| T16 | Networking    | Internal ClusterIP Service    | Done   | Required for in-cluster access  |
| T17 | Validation    | Docker push/pull test         | Done   | Confirms full registry pipeline |
| T18 | Security|add auth (htpasswd)
| T18 | Security|add TLS


