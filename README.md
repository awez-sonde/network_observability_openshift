# Network Observability in OpenShift using Minio, LokiStack, and FlowCollector

> **Environment:** OpenShift Container Platform 4.21.5 · RHCOS 9.6 · Loki Operator v6.4.3 · Network Observability Operator v1.11.0

This guide walks through deploying end-to-end network observability on an OpenShift cluster using a community Minio instance as S3-compatible object storage, LokiStack for log aggregation, and FlowCollector to capture eBPF-based network flows. The end result is a fully functional **Network Traffic** view in the OpenShift console showing real-time topology and flow data.

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Storage layer (namespace: minio-storage)                       │
│                                                                  │
│  PVC (100Gi) → Minio Deployment → Service → Route (TLS)        │
│                        │                                         │
│                   bucket: loki-data                             │
└────────────────────────┬────────────────────────────────────────┘
                         │  S3 endpoint (http://minio-service.
                         │  minio-storage.svc:9000)
┌────────────────────────▼────────────────────────────────────────┐
│  Log aggregation layer (namespace: netobserv)                   │
│                                                                  │
│  Secret: loki-s3 → LokiStack CR (lokistack-sample)             │
│          Distributor · Ingester · Gateway · Querier · Compactor │
└────────────────────────┬────────────────────────────────────────┘
                         │  LokiStack reference
┌────────────────────────▼────────────────────────────────────────┐
│  Flow collection layer (namespace: netobserv)                   │
│                                                                  │
│  FlowCollector CR → eBPF Agent (DaemonSet) → Processor (FLP)   │
│                   → Console Plugin → Network Traffic UI         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites

- An OpenShift 4.21+ cluster with `cluster-admin` access
- `oc` CLI logged in to the cluster
- A StorageClass capable of provisioning `ReadWriteOnce` PVCs — Minio uses a PVC to persist object data (S3 buckets and their contents) across pod restarts, and LokiStack also requires PVCs for its stateful components

---

## Operator installation

Two operators are required. Install them before deploying any workloads.

### Install the Loki Operator

The Loki Operator is installed in the `openshift-operators-redhat` namespace, which is the recommended namespace for Red Hat-provided operators that need cluster-wide scope.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-operators-redhat
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-operators-redhat-og
  namespace: openshift-operators-redhat
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: loki-operator
  namespace: openshift-operators-redhat
spec:
  channel: stable-6.4
  installPlanApproval: Automatic
  name: loki-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Apply with:

```bash
oc apply -f loki-operator.yaml
```

### Install the Network Observability Operator

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-netobserv-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: netobserv-og
  namespace: openshift-netobserv-operator
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: netobserv-operator
  namespace: openshift-netobserv-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: netobserv-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

Apply with:

```bash
oc apply -f netobserv-operator.yaml
```

### Verify all operators are healthy

Wait a couple of minutes for the operators to install, then confirm they all reach the `Succeeded` phase:

```bash
oc get csv -A | grep -E 'loki|netobserv'
```

Expected output:

```
NAME                                     DISPLAY                VERSION   PHASE
loki-operator.v6.4.3                     Loki Operator          6.4.3     Succeeded
network-observability-operator.v1.11.0   Network Observability  1.11.0    Succeeded
```

> **Tip:** You can also verify from the OpenShift console under **Operators → Installed Operators**. Both should show a green checkmark.

---

## Part 1 — Install Minio (community edition)

Minio serves as the S3-compatible object storage backend. We deploy it in a dedicated namespace so it remains isolated from the observability stack.

### 1.1 Create the namespace

```bash
oc new-project minio-storage
```

### 1.2 Apply all Minio resources

The following manifest creates everything in a single apply: a PVC for data persistence, a Secret for credentials, a Deployment running the Minio server, a ClusterIP Service exposing both the API and console ports, and two TLS-terminated Routes.

```yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: minio-pvc
  namespace: minio-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  volumeMode: Filesystem
---
kind: Secret
apiVersion: v1
metadata:
  name: minio-secret
  namespace: minio-storage
stringData:
  minio_root_user: minioadmin
  minio_root_password: miniopassword
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: minio
  namespace: minio-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-pvc
      containers:
        - name: minio
          image: quay.io/minio/minio:latest
          args:
            - server
            - /data
            - --console-address
            - :9090
          env:
            - name: MINIO_ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_user
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_password
          ports:
            - containerPort: 9000   # S3 API
            - containerPort: 9090   # Web console
          volumeMounts:
            - name: data
              mountPath: /data
              subPath: minio
          resources:
            limits:
              cpu: 250m
              memory: 1Gi
            requests:
              cpu: 20m
              memory: 100Mi
          readinessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            tcpSocket:
              port: 9000
            initialDelaySeconds: 30
            periodSeconds: 5
          imagePullPolicy: IfNotPresent
---
kind: Service
apiVersion: v1
metadata:
  name: minio-svc
  namespace: minio-storage
spec:
  selector:
    app: minio
  ports:
    - name: api
      port: 9000
      targetPort: 9000
    - name: ui
      port: 9090
      targetPort: 9090
  type: ClusterIP
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-api
  namespace: minio-storage
spec:
  to:
    kind: Service
    name: minio-svc
    weight: 100
  port:
    targetPort: api
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-ui
  namespace: minio-storage
spec:
  to:
    kind: Service
    name: minio-svc
    weight: 100
  port:
    targetPort: ui
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Apply with:

```bash
oc apply -f minio.yaml
```

### 1.3 Verify Minio is running

```bash
oc get pods -n minio-storage
oc get svc -n minio-storage
# NAME        TYPE        CLUSTER-IP       PORT(S)
# minio-svc   ClusterIP   172.30.156.55    9000/TCP,9090/TCP

oc get routes -n minio-storage
```

### 1.4 Create the Loki bucket

Open the Minio console via the `minio-ui` route, log in with `minioadmin / miniopassword`, and create a bucket named **`loki-data`**.

> **Screenshot:** Minio Object Browser showing the `loki-data` bucket with `index` and `network` folders after LokiStack starts writing — 253.4 MiB across 11,054 objects.

Once Loki begins ingesting flows, you will see two folders appear automatically inside `loki-data`:
- `index/` — Loki's TSDB index chunks
- `network/` — the actual network flow log chunks

---

## Part 2 — Install LokiStack

LokiStack is deployed using the Loki Operator. It requires a Secret referencing the Minio S3 endpoint before the CR can be created.

### 2.1 Create the namespace

```bash
oc new-project netobserv
```

### 2.2 Create the loki-s3 secret

This secret connects LokiStack to the Minio instance. The endpoint uses the internal cluster DNS name so traffic never leaves the cluster.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: loki-s3
  namespace: netobserv
stringData:
  access_key_id: minioadmin
  access_key_secret: miniopassword
  bucketnames: loki-data
  endpoint: http://minio-svc.minio-storage.svc:9000
  region: eu-central-1
```

> **Note:** The `region` value is required by the S3 SDK but Minio does not enforce it — any non-empty string works.

Apply with:

```bash
oc apply -f loki-s3-secret.yaml
```

> **Screenshot:** The `loki-s3` Secret in the `netobserv` namespace as seen in the OpenShift console, showing all five fields: `access_key_id`, `access_key_secret`, `bucketnames: loki-data`, `endpoint: http://minio-service.minio-storage.svc:9000`, and `region: eu-central-1`.

### 2.3 Create the LokiStack custom resource

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: lokistack-sample
  namespace: netobserv
spec:
  managementState: Managed
  size: 1x.extra-small
  storage:
    schemas:
      - effectiveDate: "2024-10-11"
        version: v13
    secret:
      name: loki-s3
      type: s3
      credentialMode: static
  storageClassName: <your-storage-class>
  hashRing:
    type: memberlist
  limits:
    global:
      queries:
        queryTimeout: 3m
  tenants:
    mode: openshift-network
```

> **Important:** Use schema version `v13` with a current `effectiveDate`. The earlier example in this repo uses `v11` from 2020 — this triggers a `StorageNeedsSchemaUpdate` warning. Using `v13` from the start avoids that warning entirely.

Apply with:

```bash
oc apply -f lokistack.yaml
```

### 2.4 Verify LokiStack health

```bash
oc get lokistack -n netobserv
```

Wait until all components are `Ready`. This typically takes 2–3 minutes as the operator provisions StatefulSets for the ingester, compactor, and index-gateway.

```bash
oc get pods -n netobserv | grep lokistack
```

Expected pods when healthy:

| Component | Replicas | Role |
|-----------|----------|------|
| `lokistack-sample-distributor` | 2 | Receives incoming log streams |
| `lokistack-sample-ingester` | 2 | Writes chunks to object storage |
| `lokistack-sample-querier` | 2 | Executes LogQL queries |
| `lokistack-sample-query-frontend` | 2 | Query sharding and caching |
| `lokistack-sample-gateway` | 2 | Multi-tenant authentication proxy |
| `lokistack-sample-index-gateway` | 2 | Serves the TSDB index |
| `lokistack-sample-compactor` | 1 | Compacts and deletes old chunks |

---

## Part 3 — Deploy FlowCollector

The FlowCollector CR is a cluster-scoped resource managed by the Network Observability Operator. It wires together the eBPF agent, flow log processor (FLP), and console plugin.

### 3.1 Create the FlowCollector

```yaml
apiVersion: flows.netobserv.io/v1beta2
kind: FlowCollector
metadata:
  name: cluster
spec:
  namespace: netobserv
  deploymentModel: Service

  agent:
    type: eBPF
    ebpf:
      sampling: 400
      cacheMaxFlows: 120000
      cacheActiveTimeout: 15s
      privileged: true
      features:
        - PacketDrop
        - UDNMapping
        - PacketTranslation
        - NetworkEvents
        - FlowRTT
      excludeInterfaces:
        - lo
      resources:
        limits:
          memory: 800Mi
        requests:
          cpu: 100m
          memory: 50Mi

  processor:
    logTypes: Flows
    consumerReplicas: 3
    resources:
      limits:
        memory: 800Mi
      requests:
        cpu: 100m
        memory: 100Mi

  loki:
    enable: true
    mode: LokiStack
    lokiStack:
      name: lokistack-sample
      namespace: netobserv
    writeBatchSize: 10485760
    writeBatchWait: 1s
    writeTimeout: 10s
    readTimeout: 30s

  consolePlugin:
    enable: true
    replicas: 2
    portNaming:
      enable: true
    quickFilters:
      - name: Applications
        filter:
          flow_layer: '"app"'
        default: true
      - name: Infrastructure
        filter:
          flow_layer: '"infra"'
      - name: Pods network
        filter:
          src_kind: '"Pod"'
          dst_kind: '"Pod"'
        default: true
      - name: Services network
        filter:
          dst_kind: '"Service"'
      - name: External ingress
        filter:
          src_subnet_label: '"",EXT:'
      - name: External egress
        filter:
          dst_subnet_label: '"",EXT:'
    resources:
      limits:
        cpu: 1000m
        memory: 2Gi
      requests:
        cpu: 100m
        memory: 100Mi

  networkPolicy:
    enable: true
    additionalNamespaces:
      - udn-poc

  prometheus:
    querier:
      enable: true
      mode: Auto
      timeout: 30s
```

Apply with:

```bash
oc apply -f flowcollector.yaml
```

### 3.2 Key configuration notes

**eBPF agent features:**

| Feature | What it captures |
|---------|-----------------|
| `PacketDrop` | Kernel-level packet drops with reason codes |
| `UDNMapping` | Flow mapping for User-Defined Networks (KubeVirt VMs) |
| `PacketTranslation` | NAT translations (DNAT/SNAT) |
| `NetworkEvents` | Network policy enforcement events |
| `FlowRTT` | Round-trip time measurements per flow |

**Sampling rate:** `400` means 1 in every 400 packets is sampled. Lower the value for more detail at the cost of higher resource usage.

**`UDNMapping`** is particularly relevant if you are running KubeVirt VMs on User-Defined Networks — it ensures VM-level traffic is correctly attributed in the topology view.

**`networkPolicy.additionalNamespaces`:** Adding `udn-poc` here allows the FlowCollector to collect flows from that namespace even if it is isolated by a NetworkPolicy.

### 3.3 Verify FlowCollector status

```bash
oc get flowcollectors
```

```
NAME      AGENT   SAMPLING (EBPF)   DEPLOYMENT MODEL   STATUS   WARNINGS
cluster   eBPF    400               Service            Ready
```

The `Ready` status means all 6 components are healthy: the eBPF DaemonSet, the FLP processor pods, the console plugin, and the network policy controller.

---

## Part 4 — Validation and the Network Traffic UI

Once the FlowCollector is `Ready`, navigate to **Observe → Network Traffic** in the OpenShift console.

The console plugin provides three views:

- **Overview** — aggregated metrics and top talkers
- **Traffic flows** — a live table of individual flow records with LogQL-powered filtering
- **Topology** — a graph of communication between namespaces, nodes, pods, or owners

> **Screenshot:** The Topology view filtered to namespace `udn-test1`, showing three KubeVirt VMs (`vm-l2-a`, `vm-l2-b`, `vm-l2-udn-bridged`) communicating within the UDN. The yellow badge on `vm-l2-b` indicates active flow data (0.2 Bps). The UDNMapping feature is what makes VM-level attribution possible here.

### Quick filter reference

The console plugin ships with pre-configured quick filters accessible from the toolbar:

| Filter | What it shows |
|--------|--------------|
| Applications | App-tier flows (`flow_layer = app`) |
| Infrastructure | Platform flows (DNS, API server, monitoring) |
| Pods network | Pod-to-pod traffic only |
| Services network | Flows destined for a Service VIP |
| External ingress | Flows arriving from outside the cluster |
| External egress | Flows leaving the cluster |

---

## Troubleshooting

**LokiStack warning: `StorageNeedsSchemaUpdate`**
The schema version `v11` is deprecated. Update the LokiStack CR to add a new schema entry:

```yaml
storage:
  schemas:
    - effectiveDate: "2020-10-11"
      version: v11
    - effectiveDate: "2024-10-11"   # add this
      version: v13
```

**FlowCollector stuck in `Pending`**
Check that the `loki-s3` secret exists in the `netobserv` namespace and that the Minio endpoint is reachable:

```bash
oc get secret loki-s3 -n netobserv
oc run -it --rm curl --image=curlimages/curl --restart=Never -- \
  curl -v http://minio-svc.minio-storage.svc:9000/minio/health/live
```

**No flows appearing in the UI**
Verify the eBPF agent DaemonSet is fully rolled out:

```bash
oc get ds -n netobserv | grep ebpf
```

Each node must have one eBPF agent pod in `Running` state. The agent requires `privileged: true` to attach eBPF programs to the kernel.

**Console plugin not loading**
Check the plugin pods:

```bash
oc get pods -n netobserv | grep console-plugin
oc logs -n netobserv -l app=netobserv-plugin
```

---

## Summary

| Component | Namespace | Key resource | Status check |
|-----------|-----------|-------------|--------------|
| Minio | `minio-storage` | Deployment `minio` | `oc get pods -n minio-storage` |
| LokiStack | `netobserv` | `LokiStack/lokistack-sample` | `oc get lokistack -n netobserv` |
| FlowCollector | cluster-scoped | `FlowCollector/cluster` | `oc get flowcollectors` |
| Console plugin | `netobserv` | Deployment `netobserv-plugin` | Observe → Network Traffic |

With this stack in place, every packet sampled by the eBPF agent flows through the FLP processor, gets indexed by LokiStack, persisted in Minio, and surfaced in the OpenShift console — giving you full network observability with no external dependencies.
