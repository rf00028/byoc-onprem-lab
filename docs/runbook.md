# BYOC CloudPrem — Bare-Metal Lab Runbook

**Document:** Lab Validation & Deviation Report
**Date:** June 17, 2026
**Author:** Ricky Fair, Datadog SE
**Reference Article:** [BYOC Logs Lab Environment](https://datadoghq.atlassian.net/wiki/spaces/GTMSEH/pages/5388141147/BYOC+Logs+Lab+Environment)
**Objective:** Validate the "BYOC On-Prem Lab" section of the article verbatim on a fresh bare-metal EC2 instance and document every deviation.

---

## Environment

| Resource | Value |
|---|---|
| Kubernetes host | m5zn.metal, Ubuntu 22.04, 300 GB gp3, us-west-1 |
| PostgreSQL host | t3.micro, Ubuntu 22.04, 20 GB, us-west-1 |
| Kubernetes version | v1.32 (kubeadm) |
| CloudPrem chart | datadog/cloudprem latest |
| Remote access | AWS SSM SendCommand only (port 22 blocked) |

---

## Installation Steps & Commands

### 1. Kubernetes Bootstrap (kubeadm)

**What:** Installs a single-node Kubernetes cluster on the bare-metal EC2 instance using kubeadm.
**Why:** EKS was unavailable due to AWS sandbox SCPs and Elastic IP quota exhaustion.

```bash
# Install container runtime
apt-get install -y containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml
systemctl restart containerd

# Install Kubernetes tools
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' \
  > /etc/apt/sources.list.d/kubernetes.list
apt-get update && apt-get install -y kubelet kubeadm kubectl

# Init cluster using PRIVATE IP (avoids hairpin kubeconfig issue)
kubeadm init --control-plane-endpoint=<PRIVATE_IP>:6443 \
  --pod-network-cidr=10.0.0.0/16 \
  --skip-phases=addon/kube-proxy

mkdir -p /root/.kube
cp /etc/kubernetes/admin.conf /root/.kube/config

# Remove control-plane taint (single-node: pods must schedule here)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

**Deviation from article:** Article uses the external kubeadm repo (`setup.sh`). We ran these steps directly via SSM to avoid SSH dependency.

---

### 2. Cilium CNI

**What:** Installs Cilium as the Container Network Interface plugin. Without a CNI, the node stays NotReady and no pods can communicate.
**Why Cilium:** eBPF-based, high performance, actively maintained. Used by default in many production Kubernetes deployments.

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update

# Critical: bare-metal requires explicit API server address
# Without these flags, Cilium init container cannot reach the API server
helm upgrade --install cilium cilium/cilium \
  --version 1.17.4 \
  --namespace kube-system \
  --set k8sServiceHost=<PRIVATE_IP> \
  --set k8sServicePort=6443

kubectl -n kube-system rollout status daemonset/cilium --timeout=180s
kubectl wait node --all --for=condition=Ready --timeout=180s
```

**Deviation from article:** Article does not document the `--set k8sServiceHost/Port` requirement. Without these flags, Cilium's init container cannot reach the API server during bootstrap on bare metal (cluster service IP `10.96.0.1` is unreachable until CNI is up — classic bootstrap deadlock).

---

### 3. local-path-provisioner (Storage)

**What:** A lightweight dynamic PersistentVolume provisioner that creates volumes backed by directories on the node's local disk. Used by default in k3s, kind, and minikube.
**Why not Longhorn:** Longhorn (specified in the article) has a webhook reconciliation deadlock on k8s 1.32+ — see Deviation 4 below.

```bash
kubectl apply -f \
  https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml

# Set as default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get storageclass
# NAME                   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer
```

---

### 4. SeaweedFS (S3-Compatible Object Store)

**What:** Distributed, S3-compatible object store. CloudPrem uses it to store indexed log data (Parquet split files).
**Why not MinIO:** MinIO's open-source repository was archived in 2024 — supply chain risk. SeaweedFS is actively maintained with a comparable S3 API.

```bash
helm repo add seaweedfs https://seaweedfs.github.io/seaweedfs/helm
helm repo update

# values file — lab-sized, local-path storage
cat > seaweedfs-values.yaml << 'EOF'
master:
  replicas: 1
  volumeSizeLimitMB: 30000
  data:
    type: persistentVolumeClaim
    size: 30G
    storageClass: local-path
volume:
  replicas: 1
  dataDirs:
    - name: data
      size: 100Gi
      type: persistentVolumeClaim
      storageClass: local-path
      maxVolumes: 100
filer:
  replicas: 1
  enablePVC: true
  data:
    type: persistentVolumeClaim
    size: 10G
    storageClass: local-path
s3:
  enabled: true
  port: 8333
  enableAuth: true
persistence:
  enabled: true
admin:
  enabled: true
EOF

kubectl create ns seaweedfs
helm upgrade --install seaweedfs seaweedfs/seaweedfs \
  --values=seaweedfs-values.yaml -n seaweedfs

kubectl rollout status statefulset/seaweedfs-master -n seaweedfs --timeout=180s
```

**Create bucket and S3 user via weed shell** (no AWS CLI on the instance):

```bash
# Exec into master pod and use weed shell
kubectl exec -n seaweedfs seaweedfs-master-0 -- sh -c "
  echo 's3.bucket.create -name byoclogs' | weed shell -master=localhost:9333
  echo 's3.configure -access_key=<KEY> -secret_key=<SECRET> \
    -user=byoclogs -actions=Read,Write,List,Tagging \
    -buckets=byoclogs -apply' | weed shell -master=localhost:9333
"

# Store credentials as k8s secret
kubectl create secret generic byoc-logs-minio-credentials \
  --from-literal AWS_ACCESS_KEY_ID="<KEY>" \
  --from-literal AWS_SECRET_ACCESS_KEY="<SECRET>" \
  -n byoclogs
```

**Deviation from article:** Article uses the SeaweedFS admin UI via `kubectl port-forward`. Since we have no direct network access to the cluster, we used `weed shell` via `kubectl exec` instead.

---

### 5. PostgreSQL 14 (Metastore)

**What:** Relational database that stores QuickWit's split catalog — metadata about every indexed log segment (file location, time range, tags, merge history).
**Runs on:** Separate t3.micro EC2 instance in the same VPC.

```bash
# On the PostgreSQL instance (via SSM)
apt-get update && apt-get install -y postgresql
systemctl enable --now postgresql

sudo -u postgres psql << 'SQL'
CREATE USER byoclogs WITH ENCRYPTED PASSWORD 'byoclogs';
CREATE DATABASE byoclogs;
GRANT ALL PRIVILEGES ON DATABASE byoclogs TO byoclogs;
ALTER DATABASE byoclogs OWNER TO byoclogs;
SQL

# Allow remote connections
PG_CONF=$(find /etc/postgresql -name postgresql.conf | head -1)
PG_HBA=$(find /etc/postgresql -name pg_hba.conf | head -1)

sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" "$PG_CONF"
echo "host  byoclogs  byoclogs  10.0.0.0/8  md5" >> "$PG_HBA"
systemctl restart postgresql
```

**Deviation from article:** Article adds `host all all 0.0.0.0/0 scram-sha-256` but doesn't note that PostgreSQL 14 creates users with `md5` auth by default. Using `scram-sha-256` in pg_hba.conf with an `md5` password hash causes authentication failure. Use `md5` in pg_hba.conf to match.

---

### 6. CloudPrem

**What:** The full Datadog CloudPrem stack: indexer, searcher, metastore, control-plane, janitor.
**How it connects to Datadog:** The control-plane initiates an outbound reverse WebSocket to `app.datadoghq.com`. No public ingress or firewall rules required.

```bash
# On the Kubernetes instance
kubectl create ns byoclogs

# Secrets
kubectl create secret generic datadog-secret \
  --from-literal api-key="<DD_API_KEY>" -n byoclogs

kubectl create secret generic byoc-logs-metastore-uri \
  --from-literal QW_METASTORE_URI="postgres://byoclogs:byoclogs@<PG_IP>:5432/byoclogs?sslmode=disable" \
  -n byoclogs
# NOTE: ?sslmode=disable is REQUIRED — see Deviation 5 below

helm repo add datadog https://helm.datadoghq.com && helm repo update

cat > datadog-values.yaml << 'EOF'
datadog:
  site: datadoghq.com
  apiKeyExistingSecret: datadog-secret

serviceAccount:
  create: true
  name: byoclogs

config:
  default_index_root_uri: s3://byoclogs/indexes
  storage:
    s3:
      endpoint: http://seaweedfs-s3.seaweedfs.svc.cluster.local:8333
      force_path_style_access: true

metastore:
  extraEnvFrom:
    - secretRef:
        name: byoc-logs-metastore-uri
    - secretRef:
        name: byoc-logs-minio-credentials

indexer:
  replicaCount: 1
  podSize: large
  persistentVolume:
    enabled: true
    storage: 50Gi
    storageClass: local-path    # NOT longhorn — see Deviation 4
  extraEnvFrom:
    - secretRef:
        name: byoc-logs-minio-credentials

searcher:
  replicaCount: 1
  podSize: large
  extraEnvFrom:
    - secretRef:
        name: byoc-logs-minio-credentials

controlPlane:
  extraEnvFrom:
    - secretRef:
        name: byoc-logs-minio-credentials

janitor:
  extraEnvFrom:
    - secretRef:
        name: byoc-logs-minio-credentials
EOF

helm upgrade --install byoclogs datadog/cloudprem \
  -f datadog-values.yaml -n byoclogs

kubectl wait --for=condition=Ready pod \
  -l app.kubernetes.io/instance=byoclogs -n byoclogs --timeout=300s
kubectl get pods -n byoclogs
```

**Verify:** Navigate to `https://app.datadoghq.com/byoc-logs` — cluster should appear as **Connected** with connection type **Reverse**.

---

### 7. Datadog Operator + Agent

**What:** The Datadog Operator manages the Agent deployment via a `DatadogAgent` custom resource. The node agent collects all pod logs and ships them to the local CloudPrem indexer.

```bash
# Install operator
helm upgrade --install datadog-operator datadog/datadog-operator -n byoclogs
kubectl rollout status deployment/datadog-operator -n byoclogs --timeout=120s

# Apply agent manifest
cat > agent-manifest.yaml << 'EOF'
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
  namespace: byoclogs
spec:
  global:
    clusterName: cloudprem
    site: datadoghq.com
    kubelet:
      tlsVerify: false
    credentials:
      apiSecret:
        secretName: datadog-secret
        keyName: api-key
    env:
      - name: DD_LOGS_CONFIG_LOGS_DD_URL
        value: http://byoclogs-cloudprem-indexer.byoclogs.svc.cluster.local:7280
      - name: DD_LOGS_CONFIG_EXPECTED_TAGS_DURATION
        value: "100000"
  features:
    logCollection:
      enabled: true
      containerCollectAll: true
    prometheusScrape:
      enabled: true
      enableServiceEndpoints: true
  override:
    nodeAgent:
      env:
        - name: DD_HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
EOF

kubectl apply -f agent-manifest.yaml

kubectl wait deployment/datadog-cluster-agent -n byoclogs \
  --for=condition=Available --timeout=180s
kubectl rollout status daemonset/datadog-agent -n byoclogs --timeout=180s
kubectl get pods -n byoclogs
```

**Expected output:**
```
datadog-agent-xxxxx              3/3     Running   0
datadog-cluster-agent-xxxxx      1/1     Running   0
datadog-operator-xxxxx           1/1     Running   0
byoclogs-cloudprem-control-plane 1/1     Running   0
byoclogs-cloudprem-indexer-0     1/1     Running   0
byoclogs-cloudprem-janitor       1/1     Running   0
byoclogs-cloudprem-metastore (×2) 1/1   Running   0
byoclogs-cloudprem-searcher-0    1/1     Running   0
```

---

## Deviations from Article

### Deviation 1 — EKS Unavailable: Pivoted to Bare Metal

**Article:** "AWS EKS Lab Environment" using `eksctl`.
**Problem:** EKS blocked by SCP in us-east-1; Elastic IP quota exhausted in us-west-1.
**Fix:** Used the article's alternate "BYOC On-Prem Lab" path with kubeadm on m5zn.metal.

---

### Deviation 2 — Longhorn Webhook Deadlock (k8s 1.32+)

**Article:** Use Longhorn for dynamic volume provisioning.
**Problem:** Longhorn's admission webhook (`validator.longhorn.io`) has `failurePolicy: Fail`. During startup, the Longhorn manager tries to create a `CRDAPIVersionSetting` CR, which calls its own webhook. The webhook has no endpoints yet (the manager pod is still initializing). The call times out, the manager crashes, and it loops. Longhorn's controller reconciliation immediately reverts any `failurePolicy: Ignore` patches — the controller always wins.
**Fix:** Replaced Longhorn with `local-path-provisioner` v0.0.31. Functionally identical for single-node labs; none of Longhorn's HA replication is useful with one node anyway.

**Article update needed:** Note that Longhorn 1.7.x and 1.8.x are broken on k8s 1.26+. Recommend local-path-provisioner for single-node labs.

---

### Deviation 3 — MinIO Replaced by SeaweedFS

**Article:** Use MinIO for S3-compatible storage.
**Problem:** MinIO's Apache-licensed open-source repository was archived in 2024. Using it poses a supply chain risk.
**Fix:** SeaweedFS provides an equivalent S3 API and is actively maintained.

**Article update needed:** Replace MinIO with SeaweedFS throughout.

---

### Deviation 4 — kubeconfig Uses Public IP; Patch Required

**Article:** kubectl works after kubeadm init.
**Problem:** `install_control_plane.sh` sets `--control-plane-endpoint` to the instance's public IP. Running kubectl on the instance (via SSM) produces `i/o timeout` because the SG doesn't allow inbound port 6443 from the instance's own public IP (no hairpin NAT).
**Fix:** Pass the private IP directly to `--control-plane-endpoint` so the kubeconfig is correct from the start. Or patch after the fact:

```bash
sed -i 's|https://<PUBLIC_IP>:6443|https://<PRIVATE_IP>:6443|' /root/.kube/config
```

**Article update needed:** Note that `--control-plane-endpoint` should use the private IP when SSH is not available and kubectl is run from within the instance via SSM.

---

### Deviation 5 — QuickWit Requires `?sslmode=disable`

**Article:** Metastore URI: `postgres://user:pass@host:5432/db` (no SSL parameter).
**Problem:** QuickWit's PostgreSQL client defaults to requiring SSL. A default PostgreSQL install has no SSL certs configured. Result: `no pg_hba.conf entry for host "...", SSL encryption`.
**Fix:** Append `?sslmode=disable` to the URI:

```
postgres://byoclogs:byoclogs@<IP>:5432/byoclogs?sslmode=disable
```

**Article update needed:** Add `?sslmode=disable` to the example connection string, or document that PostgreSQL must be configured with SSL certs if omitted.

---

### Deviation 6 — Cilium Requires k8sServiceHost/Port on Bare Metal

**Article:** Cilium install command does not include `--set k8sServiceHost` flags.
**Problem:** On bare metal, during Cilium's init phase, the cluster service IP (`10.96.0.1`) is not yet reachable. Cilium's init container tries to contact the API server via the cluster service IP and fails. The pod crashes with `x509: certificate is valid for 10.96.0.1, not 10.0.0.1`.
**Fix:**

```bash
helm install cilium cilium/cilium \
  --set k8sServiceHost=<PRIVATE_IP> \
  --set k8sServicePort=6443
```

**Article update needed:** Add these flags to the Cilium install command. They are required on all bare-metal clusters.

---

### Deviation 7 — pg_hba.conf Entry Missing / Wrong Auth Method

**Article:** Add `host all all 0.0.0.0/0 scram-sha-256` to pg_hba.conf.
**Problem A:** This entry was never added during our initial PostgreSQL install.
**Problem B:** PostgreSQL 14 creates users with `md5` password hashing by default (unless `password_encryption=scram-sha-256` is set in postgresql.conf). Using `scram-sha-256` in pg_hba.conf with an `md5` hash causes auth failure.
**Fix:**

```
host    byoclogs    byoclogs    10.0.0.0/8    md5
```

Then `systemctl reload postgresql`.

**Article update needed:** Include pg_hba.conf edit in the PostgreSQL install steps. Note the md5 vs scram-sha-256 mismatch.

---

## Verification Checklist

| Step | URL | Expected |
|---|---|---|
| Cluster connected | `app.datadoghq.com/byoc-logs` | Status: Connected, Type: Reverse |
| Logs flowing | Hover cluster → Search Logs | Logs visible within ~2 min |
| Metrics flowing | `app.datadoghq.com/metric/summary?filter=cloudprem` | cloudprem.* metrics present |

---

## Troubleshooting

### AWS STS token expired
```bash
# Paste fresh export block from AWS SSO portal, then:
aws configure set aws_access_key_id     "$AWS_ACCESS_KEY_ID"     --profile byoc
aws configure set aws_secret_access_key "$AWS_SECRET_ACCESS_KEY" --profile byoc
aws configure set aws_session_token     "$AWS_SESSION_TOKEN"     --profile byoc
aws sts get-caller-identity --profile byoc   # verify
```

### Metastore CrashLoopBackOff
```bash
kubectl logs -n byoclogs deployment/byoclogs-cloudprem-metastore --tail=10
# If "SSL encryption" → check ?sslmode=disable in URI
# If "no pg_hba.conf entry" → add host entry and reload postgres
```

### Indexer Pending (VolumeBinding failed)
```bash
kubectl describe pod byoclogs-cloudprem-indexer-0 -n byoclogs | grep -A5 Events
# If "failed calling webhook validator.longhorn.io" → delete lingering webhook:
kubectl delete validatingwebhookconfiguration longhorn-webhook-validator --ignore-not-found
kubectl delete mutatingwebhookconfiguration longhorn-webhook-mutator --ignore-not-found
```

### Pods stuck in Pending
```bash
# Check node taint
kubectl describe node | grep Taint
# If control-plane taint present:
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
