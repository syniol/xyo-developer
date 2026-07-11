# ☸️ XYO Enrichment Kubernetes Deployment

This folder contains standard Kubernetes manifests for running the XYO Enrichment Service at scale.

---

## Deployment Architecture

The manifests deploy the service inside a dedicated namespace (`xyo`), structuring components as follows:
- **`xyo-postgres`**: Cached data backend (1 replica).
- **`xyo-oracle`** & **`xyo-yoda`**: Internal machine learning and heuristic engines (1 replica each).
- **`xyo-enrichment`**: Processing layer (2 replicas, scales horizontally).
- **`xyo-gateway`**: Unified public endpoint (2 replicas, scales horizontally).
- **`xyo-logos-pvc`**: Shared storage volume (ReadWriteMany) to persist and share downloaded logos between `enrichment` and `gateway`.

---

## Installation Steps

### Step 1: Create the Namespace
```bash
kubectl create namespace xyo
```

### Step 2: Configure Image Pull Secrets
Since XYO containers are hosted on a private registry (`cr.syniol.com`), configure Kubernetes to authenticate with your registry credentials:
```bash
kubectl create secret docker-registry syniol-registry-secret \
  --docker-server=cr.syniol.com \
  --docker-username="<your-client-id>" \
  --docker-password="<your-client-secret>" \
  --namespace=xyo
```

### Step 3: Configure the License Secret
Store your activation license key securely:
```bash
kubectl create secret generic xyo-license-secret \
  --from-literal=license-key="your-activation-license-key-here" \
  --namespace=xyo
```

### Step 4: Provision Persistent Storage
Deploy the PersistentVolumeClaim for logos. Ensure your cluster has a default `StorageClass` configured, or edit [pv-pvc.yaml](file:///Users/hadi/dev/start-ups/xyo/onboarding/kubernetes/pv-pvc.yaml) to reference a specific `StorageClass` (ideally SSD-backed):
```bash
kubectl apply -f pv-pvc.yaml
```

### Step 5: Deploy Services
Apply the manifests in order:
```bash
# 1. Start the database
kubectl apply -f postgres.yaml

# 2. Start internal processing components
kubectl apply -f oracle.yaml
kubectl apply -f yoda.yaml

# 3. Start the enrichment coordinator
kubectl apply -f enrichment.yaml

# 4. Start the gateway entrypoint
kubectl apply -f gateway.yaml
```

---

## Verifying Deployment

Ensure all pods are in the `Running` state:
```bash
kubectl get pods -n xyo
```

Ensure services are configured properly:
```bash
kubectl get svc -n xyo
```

### Direct Testing (Port Forward)
To verify the deployment from your local machine, port-forward the Gateway service:
```bash
kubectl port-forward svc/xyo-gateway 8080:8080 -n xyo
```

Now, send a sample request:
```bash
curl -X POST http://localhost:8080/v1/enrich \
  -H "Content-Type: application/json" \
  -d '{
    "description": "AMZN Mktp US*Amzn.com/bill WA",
    "countryCode": "US"
  }'
```

---

## Production Recommendations

1. **Storage Class**: Use a high-IOPS SSD class (e.g., `gp3` on AWS, `premium-rwo` on GCP) for `xyo-logos-pvc` to minimize logo asset retrieval latency.
2. **Ingress**: Map an Ingress controller to service `xyo-gateway` on port `8080` (HTTP). Enable TLS termination at the Ingress controller.
3. **Database**: For production, swap `postgres.yaml` with a managed database service (e.g. AWS RDS PostgreSQL, GCP Cloud SQL) and update `XYO_DB_DSN` in `gateway.yaml`.
