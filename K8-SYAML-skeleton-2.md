Below are **fully populated, production‑grade skeletons** for every item you listed.
They show *every* common field you are likely to set in the `spec:` (or equivalent) for each resource on a modern Kubernetes v1.30 cluster.

> **Tip** Copy the snippet you need, then delete or tweak fields to match your workload.
> Default/optional fields are commented so you can remove them quickly.

---

## 1  Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-pod
  labels:
    app: demo
  annotations:
    backup.velero.io/backup-volumes: data
spec:
  containers:
  - name: web
    image: nginx:1.27
    ports:
    - containerPort: 80
    env:
    - name: LOG_LEVEL
      value: info
    resources:
      requests:
        cpu: 250m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      timeoutSeconds: 2
    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
  initContainers:
  - name: init-permissions
    image: busybox
    command: ["sh", "-c", "chmod 777 /work"]
    volumeMounts:
    - name: data
      mountPath: /work
  restartPolicy: Always          # OnFailure | Never
  serviceAccountName: app-sa
  securityContext:
    runAsNonRoot: true
    fsGroup: 2000
  nodeSelector:
    disktype: ssd
  tolerations:
  - key: "environment"
    operator: "Equal"
    value: "dev"
    effect: "NoSchedule"
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: "kubernetes.io/hostname"
        labelSelector:
          matchLabels:
            app: demo
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: demo-pvc
```

---

## 2  Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  labels:
    app: demo
spec:
  type: LoadBalancer          # ClusterIP | NodePort | ExternalName
  selector:
    app: demo
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  sessionAffinity: None       # ClientIP
  externalTrafficPolicy: Local
```

---

## 3  Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-ns
  labels:
    environment: dev
```

---

## 4  ReplicationController (legacy)

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: legacy-rc
spec:
  replicas: 3
  selector:
    app: legacy
  template:
    metadata:
      labels:
        app: legacy
    spec:
      containers:
      - name: web
        image: nginx:1.27
```

---

## 5  ReplicaSet

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: demo-rs
spec:
  replicas: 4
  selector:
    matchLabels:
      app: demo
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: web
        image: nginx:1.27
```

---

## 6  DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      serviceAccountName: log-agent-sa
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
```

---

## 7  Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deploy
spec:
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  progressDeadlineSeconds: 600
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: web
        image: nginx:1.27
        imagePullPolicy: IfNotPresent
```

---

## 8  StatefulSet

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
  updateStrategy:
    type: RollingUpdate
  podManagementPolicy: OrderedReady   # Parallel
  template:
    metadata:
      labels:
        app: db
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
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
      storageClassName: gp3
```

---

## 9  ConfigMap & Secret

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: info
  WELCOME_MSG: "Hello, Demo!"

---
# Secret (base64 in `data`, plain text in `stringData`)
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

## 10  Ingress (for any controller)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
    nginx.ingress.kubernetes.io/rewrite-target: /
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

## 11  PersistentVolume & PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 10Gi
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  persistentVolumeReclaimPolicy: Retain
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
  namespace: demo-ns
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3
```

---

## 12  Helm Chart Skeleton

```
demo-chart/
├── Chart.yaml            # apiVersion: v2, name, version, appVersion...
├── values.yaml           # replicaCount: 2, image: nginx, service.type: ClusterIP
└── templates/
    ├── _helpers.tpl
    ├── deployment.yaml   # {{ .Values.image.repository }}:{{ .Values.image.tag }}
    ├── service.yaml
    └── ingress.yaml
```

*`deployment.yaml` template excerpt:*

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "demo.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "demo.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "demo.name" . }}
    spec:
      containers:
      - name: web
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

---

## 13  RBAC (Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccount)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: demo-ns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: demo-ns
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
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
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-list
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: namespace-list-binding
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: demo-ns
roleRef:
  kind: ClusterRole
  name: namespace-list
  apiGroup: rbac.authorization.k8s.io
```

---

### How to proceed

* **Copy** the snippet for the component you’re creating.
* **Prune** any fields your workload doesn’t need.
* **Template** them (with Helm/Kustomize) for multiple environments.

Feel free to ask for deeper dives (e.g., advanced probes, canary rollout strategies, volume‑type comparisons) or to see live examples with `kubectl` commands.
