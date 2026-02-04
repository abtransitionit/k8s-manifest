
# Cimreg
## Definition
- a contenerized application
- an in-cluster private container image registry)
- uses 
  - the CSI `OpenEBS` hostPath feature
  - the docker `registry:2`


## Terminomogy
|Term|Meaning|
|-|-|
|Cimreg|**C**ontainer **Im**age **Reg**istry|
|EBS|**E**lastic **B**lock **S**torage|





# Design

|Key|Value|comment|
|-|-|-|
|**Application name**| cimreg
|**Image**|registry:2|
|**Ns**|cimreg
|**SC**|cimreg-sc|
|**PVC**|cimreg-pvc|
|**Access mode**|ReadWriteOnce



# If pbs with pvc
```sh
sudo mkdir -p /var/lib/cimreg-local
sudo chmod 755 /var/lib/cimreg-local
```
# Yaml
---
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: registry
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: openebs-hostpath
  resources:
    requests:
      storage: 50Gi
```
---

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          env:
            - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
              value: /var/lib/registry
          volumeMounts:
            - name: registry-storage
              mountPath: /var/lib/registry
      volumes:
        - name: registry-storage
          persistentVolumeClaim:
            claimName: registry-data
```
meaning:

* PVC mounted at `/var/lib/registry`
* single replica (important with hostpath)

---

Private means **ClusterIP only**.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  type: ClusterIP
  selector:
    app: registry
  ports:
    - port: 5000
      targetPort: 5000
```
meaning:

* Private means **ClusterIP only**.


# Test
**Test from inside the cluster**

```bash
kubectl run test --rm -it \
  --image=busybox \
  --restart=Never \
  --namespace=registry \
  -- sh
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

**Phase**
| id| Yaml| Description|
|-|-|-|
| 00| `00-ns.yaml`| `Ns:cimreg`|
| 01| `01-cm.yaml`| `CM:cimreg-cm` storing the host path variable CIMREG_HOSTPATH|
| 02| `02-prep.yaml`| `CM:cimreg-ds` create `CIMREG_HOSTPATH` on all nodes|
| 03| `03-sc.yaml`| `SC:cimreg-sc` (local hostPath)|
| 04| `04-pvc.yaml`| `PVC:cimreg-pvc` using `SC:cimreg-sc`|
| 05| `05-dep.yaml`| `Deploy:cimreg-deploy`|Deployment for Docker registry `registry:3.0.0`|

|phase|purpose|comment|
|-|-|-|
|2|Persistent Storage|PVC + Storage validation
|3|Configuration|ConfigMap or Secret(registry config.yml)
|4|Deployment|Single pod registry
|5|Service|ClusterIP first
|6|External Access|Ingress / TLS / auth
|7|Hardening|S• ecurityContext<br>• NetworkPolicy<br>• Resource limits<br>• PodDisruptionBudget

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
