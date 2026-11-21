# End-to-End Examples Deployment Guide (1 to 12)
**Domain:** `aiguru.website` | **AWS Region:** `eu-north-1` | **EKS Cluster:** `staging-demo` | **Account:** `619544673475`

This guide explains how to deploy, test, and verify each example (1 to 12) on your Amazon EKS cluster.

---

## Architecture & Subdomain Map

| Example | Feature | Access Protocol / Subdomain | Backend / Controller |
| :--- | :--- | :--- | :--- |
| **1-example** | RBAC Viewer Role | `kubectl auth can-i ... --as developer` | Kubernetes RBAC (ClusterRole) |
| **2-example** | RBAC Admin Role | `kubectl auth can-i ... --as manager` | Kubernetes RBAC (ClusterRoleBinding) |
| **3-example** | Horizontal Pod Autoscaler (HPA) | Internal ClusterIP (`myapp:8080`) | Kubernetes Metrics Server |
| **4-example** | Cluster Autoscaler (CA) | Scaled Pod Replicas | Cluster Autoscaler + EC2 ASG |
| **5-example** | AWS Network Load Balancer (NLB) | `http://<NLB-DNS>:80` & `:8080` | AWS Load Balancer Controller |
| **6-example** | AWS Application Load Balancer (ALB) | `http://ex6.aiguru.website` | AWS Load Balancer Controller (ALB) |
| **7-example** | AWS ALB with SSL / HTTPS & ACM | `https://ex7.aiguru.website` | AWS Load Balancer Controller + ACM |
| **8-example** | Ingress NGINX Controller (HTTP) | `http://ex8.aiguru.website` | Ingress-NGINX behind AWS NLB |
| **9-example** | Automated SSL/TLS with Let's Encrypt | `https://ex9.aiguru.website` | Cert-Manager + Let's Encrypt ACME |
| **10-example**| StatefulSet with Amazon EBS | Persistent Volume (ext4, ReadWriteOnce) | AWS EBS CSI Driver (`gp2`) |
| **11-example**| Shared File Storage with Amazon EFS | Multi-Pod Shared Storage (ReadWriteMany)| AWS EFS CSI Driver (`efs`) |
| **12-example**| AWS Secrets Store CSI Driver | Sync Secrets Manager into Pod & K8s Secret| Secrets Store CSI Driver + IRSA |

---

## 1-example: Read-Only RBAC (Developer)

Grants view-only permissions (get, list, watch) across pods, deployments, services, configmaps, and secrets to users in the `my-viewer` group (such as `developer`).

### Deploy:
```bash
kubectl apply -f 1-example/
```

### Verify:
```bash
# Check if developer can view pods (Should be YES):
kubectl auth can-i get pods --as developer

# Check if developer can delete pods (Should be NO):
kubectl auth can-i delete pods --as developer

# Check if developer can create deployments (Should be NO):
kubectl auth can-i create deployments --as developer
```

---

## 2-example: Cluster Admin RBAC (Manager)

Grants full cluster administrator access (`cluster-admin`) to users in the `my-admin` group (such as `manager`).

### Deploy:
```bash
kubectl apply -f 2-example/
```

### Verify:
```bash
# Check if manager can create any resource (Should be YES):
kubectl auth can-i create pods --as manager
kubectl auth can-i delete namespaces --as manager
```

---

## 3-example: Horizontal Pod Autoscaler (HPA)

Scales pods automatically from 1 to 5 replicas based on CPU utilization (> 80%) or Memory utilization (> 70%) using Metrics Server.

### Deploy:
```bash
kubectl apply -f 3-example/
```

### Verify:
```bash
kubectl get hpa -n 3-example
kubectl get deployment -n 3-example
kubectl top pods -n 3-example
```

---

## 4-example: Cluster Autoscaler (CA)

Deploys 5 replicas requesting 500m CPU each (total 2.5 CPU cores). If existing worker nodes don't have enough free CPU, Cluster Autoscaler automatically scales out the AWS EC2 Auto Scaling Group.

### Deploy:
```bash
kubectl apply -f 4-example/
```

### Verify:
```bash
kubectl get pods -n 4-example -o wide
kubectl get nodes
kubectl logs -n kube-system deployment/cluster-autoscaler --tail=50
```

---

## 5-example: AWS Network Load Balancer (NLB)

Provisions a Layer 4 AWS Network Load Balancer via AWS Load Balancer Controller using IP targets directly routing to Pod IPs.

### Deploy:
```bash
kubectl apply -f 5-example/
```

### Verify:
```bash
# Get the external NLB DNS:
kubectl get svc -n 5-example myapp

# Test in browser or curl:
curl http://<NLB-DNS-NAME>
curl http://<NLB-DNS-NAME>/health
```

---

## 6-example: AWS Application Load Balancer (ALB Ingress)

Provisions an AWS Application Load Balancer routing HTTP traffic to `ex6.aiguru.website`.

### DNS Setup:
1. In your DNS provider (e.g., Route 53, GoDaddy, Cloudflare), add a **CNAME** record:
   - **Name / Host:** `ex6` (or `ex6.aiguru.website`)
   - **Value / Target:** The ALB DNS name from `kubectl get ingress -n 6-example myapp`

### Deploy:
```bash
kubectl apply -f 6-example/
```

### Verify:
```bash
kubectl get ingress -n 6-example myapp
curl -H "Host: ex6.aiguru.website" http://<ALB-DNS-NAME>/
# Or directly once DNS has propagated:
curl http://ex6.aiguru.website/
```

---

## 7-example: AWS ALB Ingress with HTTPS & ACM Certificate

Provisions an ALB with SSL redirection (HTTP:80 -> HTTPS:443) terminating TLS with an AWS Certificate Manager (ACM) certificate.

### Prerequisites:
1. Request or import an ACM Certificate for `*.aiguru.website` or `ex7.aiguru.website` in region **`eu-north-1`**.
2. Copy your certificate ARN and update line 11 in `7-example/3-ingress.yaml`:
   ```yaml
   alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-north-1:619544673475:certificate/<YOUR_CERT_ID>
   ```

### DNS Setup:
- Add a CNAME record for `ex7.aiguru.website` pointing to the ALB DNS name.

### Deploy:
```bash
kubectl apply -f 7-example/
```

### Verify:
```bash
kubectl get ingress -n 7-example myapp
curl -k https://ex7.aiguru.website/
```

---

## 8-example: Ingress NGINX Controller

Routes traffic through the high-performance Ingress NGINX controller deployed via Helm in the `ingress` namespace behind an AWS Network Load Balancer.

### DNS Setup:
1. Get the NGINX Ingress Load Balancer DNS:
   ```bash
   kubectl get svc -n ingress external-ingress-nginx-controller
   ```
2. Add a CNAME record:
   - **Host:** `ex8`
   - **Target:** `<NGINX-NLB-DNS-NAME>`

### Deploy:
```bash
kubectl apply -f 8-example/
```

### Verify:
```bash
kubectl get ingress -n 8-example myapp
curl -H "Host: ex8.aiguru.website" http://<NGINX-NLB-DNS-NAME>/
```

---

## 9-example: Automated SSL/TLS with Cert-Manager & Let's Encrypt

Automatically requests, validates (via ACME HTTP-01 challenge), and renews a free trusted SSL/TLS certificate from Let's Encrypt for `ex9.aiguru.website`.

### DNS Setup (Crucial for ACME Validation):
- Add a CNAME record:
  - **Host:** `ex9`
  - **Target:** The NGINX Ingress Load Balancer DNS (`kubectl get svc -n ingress external-ingress-nginx-controller`).
  *(Let's Encrypt requires this DNS to resolve publicly to complete HTTP-01 challenge).*

### Deploy:
```bash
# 1. Create the ACME ClusterIssuer:
kubectl apply -f 9-example/0-cluster-issuer.yaml

# 2. Deploy the application, service, and ingress:
kubectl apply -f 9-example/1-namespace.yaml
kubectl apply -f 9-example/2-deployment.yaml
kubectl apply -f 9-example/3-service.yaml
kubectl apply -f 9-example/4-ingress.yaml
```

### Verify:
```bash
# Check certificate status (Wait 1-2 minutes for READY = True):
kubectl get certificate -n 9-example
kubectl describe certificate -n 9-example ex9-aiguru-website-tls

# Test secure HTTPS in your browser:
https://ex9.aiguru.website
```

---

## 10-example: StatefulSet with Amazon EBS (EBS CSI Driver)

Dynamically provisions an AWS EBS volume (GP2) and attaches it to a StatefulSet pod (`myapp-0`) at mount path `/data`.

### Deploy:
```bash
kubectl apply -f 10-example/
```

### Verify:
```bash
kubectl get statefulset -n 10-example
kubectl get pvc -n 10-example
kubectl get pv

# Verify the EBS volume is mounted inside the pod:
kubectl exec -it myapp-0 -n 10-example -- df -h /data
```

---

## 11-example: Shared File Storage with Amazon EFS (EFS CSI Driver)

Creates a ReadWriteMany PersistentVolumeClaim backed by Amazon EFS. Two distinct pods (`myapp-1` and `myapp-2`) mount the shared filesystem at `/data` simultaneously and write timestamps to `/data/out1.txt`.

### Deploy:
```bash
kubectl apply -f 11-example/
```

### Verify:
```bash
kubectl get pvc -n 11-example
kubectl get pods -n 11-example

# Verify both pods are writing to the exact same shared file:
kubectl exec -it myapp-1 -n 11-example -- cat /data/out1.txt
kubectl exec -it myapp-2 -n 11-example -- cat /data/out1.txt
```

---

## 12-example: AWS Secrets Store CSI Driver (Secrets Manager)

Fetches a secret from AWS Secrets Manager in `eu-north-1` using IAM Roles for Service Accounts (IRSA), mounts it as a file inside the container, and automatically syncs it to a native Kubernetes Secret.

### Step 1: Create the Secret in AWS Secrets Manager:
```bash
aws secretsmanager create-secret \
  --name staging/myapp-secret-v4 \
  --secret-string '{"username":"admin","password":"SuperSecretPassword123"}' \
  --region eu-north-1
```

### Step 2: Deploy to Kubernetes:
```bash
kubectl apply -f 12-example/
```

### Step 3: Verify:
```bash
# 1. Verify the pod is running:
kubectl get pods -n 12-example

# 2. Check the mounted secret file inside the container:
kubectl exec -it deployment/myapp -n 12-example -- ls -la /mnt/secrets

# 3. Check the synced Kubernetes Secret:
kubectl get secret myapp-k8s-secret -n 12-example -o yaml

# 4. Check the injected environment variables:
kubectl exec -it deployment/myapp -n 12-example -- env | grep MY_
```
