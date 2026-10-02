# MongoDB & Mongo Express on Linode Kubernetes (LKE) with NGINX Ingress

Production-ready deployment of a high-availability MongoDB ReplicaSet using Bitnami Helm charts, paired with a Mongo Express web administrative UI exposed through an NGINX Ingress Controller on Linode Kubernetes Engine (LKE).

---

## Architecture Overview

```text
[ Client Browser ]
        │
        │ HTTP (Port 80)
        ▼
[ Linode NodeBalancer ] (External IP: e.g., 194.195.115.185)
        │
        ▼
[ NGINX Ingress Controller ] (IngressClass: nginx)
        │
        │ Ingress Rule (Host: <external-ip>.ip.linodeusercontent.com)
        ▼
[ ClusterIP Service: mongo-express-service ] (Port 8081)
        │
        ▼
[ Pod: mongo-express ]
        │
        │ Internal DB Query (Port 27017)
        ▼
[ Headless Service: mongodb-headless ]
        │
        ├──► Pod: mongodb-0 (Primary)
        ├──► Pod: mongodb-1 (Secondary)
        └──► Pod: mongodb-2 (Secondary)
                 ▲
                 └── Managed via Bitnami StatefulSet (Storage: linode-block-storage)
```

---

## Project Structure

```text
.
├── helm-mongo-values.yaml    # Custom Helm values for Bitnami MongoDB ReplicaSet
├── mongo-express.yaml         # Mongo Express Deployment & ClusterIP Service
├── ingress.yaml               # Ingress routing rules for Mongo Express
└── README.md                  # Project documentation
```

---

## Prerequisites

- **Kubernetes Cluster**: Linode Kubernetes Engine (LKE) or any managed Kubernetes cloud cluster.
- **CLI Tools Installed Locally**:
  - `kubectl` configured with cluster access (`kubeconfig.yaml`).
  - `helm` (v3+).

> **Note for WSL2 Users:** If Helm hangs while pulling charts with a `context deadline exceeded` error, lower your WSL network interface MTU to prevent packet dropping:
> ```bash
> sudo ip link set dev eth0 mtu 1350
> ```

---

## Deployment Steps

### 1. Install the NGINX Ingress Controller
Install the official NGINX Ingress Controller using Helm:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
```

Wait until Linode provisions the external IP for the controller:

```bash
kubectl get svc -l app.kubernetes.io/name=ingress-nginx -w
```

Note the `EXTERNAL-IP` (for example, `194.195.115.185`) and its default reverse DNS hostname (`194-195-115-185.ip.linodeusercontent.com`).

---

### 2. Deploy MongoDB ReplicaSet via Helm
Add the Bitnami repository and deploy a 3-node replica set using `helm-mongo-values.yaml`:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm install mongodb bitnami/mongodb -f helm-mongo-values.yaml
```

Verify that all pods in the StatefulSet and Arbiter are running:

```bash
kubectl get pods -l app.kubernetes.io/name=mongodb
```

---

### 3. Deploy Mongo Express
Deploy the Mongo Express web UI and its corresponding `ClusterIP` service:

```bash
kubectl apply -f mongo-express.yaml
```

The Mongo Express pod automatically extracts the admin password from the `mongodb` Secret created by Bitnami:
- **Connection URL**: `mongodb-0.mongodb-headless:27017`
- **Username**: `root`
- **Password**: Loaded dynamically via `secretKeyRef` (`mongodb-root-password`)

---

### 4. Configure & Apply Ingress Routing
1. In `ingress.yaml`, ensure `spec.rules.host` matches your Linode external IP domain name and `ingressClassName` is set to `nginx`:
   ```yaml
   spec:
     ingressClassName: nginx
     rules:
     - host: <YOUR-LINODE-IP-HOSTNAME>.ip.linodeusercontent.com
   ```
2. Apply the Ingress resource:
   ```bash
   kubectl apply -f ingress.yaml
   ```

---

## Verification

Check that all components are healthy:

```bash
# Check Ingress rules and hosts
kubectl get ingress

# Check all workloads and services
kubectl get pods,svc,statefulset
```

Open your browser and navigate to:
```text
http://<YOUR-LINODE-IP-HOSTNAME>.ip.linodeusercontent.com
```

You should see the Mongo Express administrative dashboard connected to the MongoDB ReplicaSet.

---

## Teardown

To delete all provisioned resources and release Linode cloud resources (NodeBalancers and Block Storage volumes):

```bash
kubectl delete -f ingress.yaml
kubectl delete -f mongo-express.yaml
helm uninstall mongodb
helm uninstall nginx-ingress
```