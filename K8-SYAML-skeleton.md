Below are concise reference tables—one per component—showing the **current, stable `apiVersion` you would typically use on a Kubernetes v1.30‑plus cluster** and the key fields you place under `spec:` (or other top‑level keys) when authoring a manifest.  For each table I also include a **minimal YAML skeleton** as a quick copy‑paste starting point.

---

### 1. Pod

| Key          | Typical value / purpose                                                                                                  | Notes                          |
| ------------ | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------ |
| `apiVersion` | **`v1`**                                                                                                                 | Core resource, never leaves v1 |
| `kind`       | `Pod`                                                                                                                    |                                |
| `metadata`   | `name`, `labels`, `annotations`                                                                                          |                                |
| `spec`       | `containers`, `initContainers`, `volumes`, `restartPolicy`, `nodeSelector`, `tolerations`, `affinity`, `securityContext` |                                |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    ports:
        - containerPort: 80
```

---

### 2. Service

| Key          | Typical value / purpose                                                                                                           | Notes |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------- | ----- |
| `apiVersion` | **`v1`**                                                                                                                          |       |
| `kind`       | `Service`                                                                                                                         |       |
| `metadata`   | `name`, `labels`, `annotations`                                                                                                   |       |
| `spec`       | `type` (`ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`), `selector`, `ports`, `sessionAffinity`, `externalTrafficPolicy` |       |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
spec:
  type: ClusterIP
  selector:
    app: demo
  ports:
  - port: 80
    targetPort: 8080
```

---

### 3. Namespace

| Key          | Typical value / purpose                           | Notes |
| ------------ | ------------------------------------------------- | ----- |
| `apiVersion` | **`v1`**                                          |       |
| `kind`       | `Namespace`                                       |       |
| `metadata`   | `name`, `labels`                                  |       |
| *No `spec`*  | Quotas and limits are created as separate objects |       |

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-ns
```

---

### 4. ReplicationController

> Generally replaced by ReplicaSet/Deployment, but still present.

| Key          | Typical value / purpose                             | Notes |
| ------------ | --------------------------------------------------- | ----- |
| `apiVersion` | **`v1`**                                            |       |
| `kind`       | `ReplicationController`                             |       |
| `metadata`   | `name`, `labels`                                    |       |
| `spec`       | `replicas`, `selector`, `template` *(Pod template)* |       |

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: legacy-rc
spec:
  replicas: 3
  selector:
    app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: app
        image: nginx
```

---

### 5. ReplicaSet

| Key          | Typical value / purpose            | Notes |
| ------------ | ---------------------------------- | ----- |
| `apiVersion` | **`apps/v1`**                      |       |
| `kind`       | `ReplicaSet`                       |       |
| `metadata`   | `name`, `labels`                   |       |
| `spec`       | `replicas`, `selector`, `template` |       |

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: demo-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: app
        image: nginx
```

---

### 6. DaemonSet

| Key          | Typical value / purpose                                     | Notes |
| ------------ | ----------------------------------------------------------- | ----- |
| `apiVersion` | **`apps/v1`**                                               |       |
| `kind`       | `DaemonSet`                                                 |       |
| `metadata`   | `name`, `labels`                                            |       |
| `spec`       | `selector`, `template`, `updateStrategy`, `minReadySeconds` |       |

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: agent
        image: fluentd
```

---

### 7. Deployment

| Key          | Typical value / purpose                                                                           | Notes |
| ------------ | ------------------------------------------------------------------------------------------------- | ----- |
| `apiVersion` | **`apps/v1`**                                                                                     |       |
| `kind`       | `Deployment`                                                                                      |       |
| `metadata`   | `name`, `labels`                                                                                  |       |
| `spec`       | `replicas`, `selector`, `template`, `strategy`, `revisionHistoryLimit`, `progressDeadlineSeconds` |       |

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: app
        image: nginx
```

---

### 8. StatefulSet

| Key          | Typical value / purpose                                                                                            | Notes |
| ------------ | ------------------------------------------------------------------------------------------------------------------ | ----- |
| `apiVersion` | **`apps/v1`**                                                                                                      |       |
| `kind`       | `StatefulSet`                                                                                                      |       |
| `metadata`   | `name`, `labels`                                                                                                   |       |
| `spec`       | `serviceName`, `replicas`, `selector`, `template`, `volumeClaimTemplates`, `updateStrategy`, `podManagementPolicy` |       |

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:16
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

---

### 9. ConfigMap & Secret

| Resource      | `apiVersion` | Key fields                                                      |
| ------------- | ------------ | --------------------------------------------------------------- |
| **ConfigMap** | `v1`         | `data`, `binaryData`                                            |
| **Secret**    | `v1`         | `type` (`Opaque`, `kubernetes.io/tls`, …), `data`, `stringData` |

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  username: admin
  password: supersecret
```

---

### 10. Ingress (Controller)

*The manifest you write is an **Ingress** object; the actual “controller” is installed separately (Nginx, Traefik, etc.).*

| Key          | Typical value / purpose                                       | Notes |
| ------------ | ------------------------------------------------------------- | ----- |
| `apiVersion` | **`networking.k8s.io/v1`**                                    |       |
| `kind`       | `Ingress`                                                     |       |
| `metadata`   | `name`, `annotations` (class, cert‑manager, etc.)             |       |
| `spec`       | `ingressClassName`, `tls`, `rules` (`host`, `http` → `paths`) |       |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["demo.example.com"]
    secretName: demo-tls
  rules:
  - host: demo.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo-svc
            port:
              number: 80
```

---

### 11. Volumes (core types)

Volumes are fields **inside a Pod template**, not standalone resources, but two persistent objects do have their own manifests:

| Resource                        | `apiVersion` | Key `spec` fields                                                                                                             |
| ------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| **PersistentVolume (PV)**       | `v1`         | `capacity`, `accessModes`, `storageClassName`, `persistentVolumeReclaimPolicy`, `hostPath` / `nfs` / `awsElasticBlockStore` … |
| **PersistentVolumeClaim (PVC)** | `v1`         | `accessModes`, `resources.requests.storage`, `storageClassName`, `volumeMode`                                                 |

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
```

Inside a pod:

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: data
```

---

### 12. Helm Chart skeleton

Helm isn’t a single Kubernetes object; instead you create a **Chart directory**:

| File          | Purpose                                                                                   |
| ------------- | ----------------------------------------------------------------------------------------- |
| `Chart.yaml`  | Chart metadata (`apiVersion: v2`, `name`, `version`, `appVersion`, `description`, `type`) |
| `values.yaml` | Default configuration values                                                              |
| `templates/`  | YAML templates (`deployment.yaml`, `service.yaml`, `_helpers.tpl`, etc.)                  |

`Chart.yaml` minimal:

```yaml
apiVersion: v2
name: demo-chart
description: A demo Helm chart
version: 0.1.0
appVersion: "1.0.0"
type: application
```

A template uses Go templating:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "demo.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  ...
```

---

### 13. RBAC

Four core objects plus ServiceAccount:

| Resource               | `apiVersion`                   | Key `spec` (or rules)                                         |
| ---------------------- | ------------------------------ | ------------------------------------------------------------- |
| **Role**               | `rbac.authorization.k8s.io/v1` | `rules` (verbs, apiGroups, resources, resourceNames)          |
| **ClusterRole**        | `rbac.authorization.k8s.io/v1` | Same as Role, cluster‑wide                                    |
| **RoleBinding**        | `rbac.authorization.k8s.io/v1` | `subjects`, `roleRef`                                         |
| **ClusterRoleBinding** | `rbac.authorization.k8s.io/v1` | Same as RoleBinding, cluster‑wide                             |
| **ServiceAccount**     | `v1`                           | `secrets`, `imagePullSecrets`, `automountServiceAccountToken` |

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: demo-ns
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: demo-ns
subjects:
- kind: ServiceAccount
  name: app-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

