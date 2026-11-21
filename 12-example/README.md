# 12-example: AWS Secrets Store CSI Driver

This example securely injects secrets from **AWS Secrets Manager** directly into Kubernetes Pods and syncs them to native Kubernetes Secrets without storing plaintext secrets in Git.

## Architecture

- **AWS Secrets Manager (eu-north-1):** Stores `staging/myapp-secret-v4` containing JSON keys `username` and `password`.
- **IAM Role for Service Account (IRSA):** `staging-demo-myapp-secrets` (ARN: `arn:aws:iam::619544673475:role/staging-demo-myapp-secrets`) created by `21-secrets-store-csi-driver.tf`.
- **Kubernetes ServiceAccount:** Annotated with the IAM Role ARN.
- **SecretProviderClass:** Defines which AWS Secrets Manager secret to fetch and how to map keys to environment variables and Kubernetes secrets.

---

## Step 1: Create the Secret in AWS Secrets Manager

Before deploying the Kubernetes manifests, create the secret in AWS Secrets Manager in region `eu-north-1`:

```bash
aws secretsmanager create-secret \
  --name staging/myapp-secret-v4 \
  --description "Database credentials for myapp" \
  --secret-string '{"username":"admin","password":"SuperSecretPassword123"}' \
  --region eu-north-1
```

*(If you already created it, you can update it with:)*
```bash
aws secretsmanager put-secret-value \
  --secret-id staging/myapp-secret-v4 \
  --secret-string '{"username":"admin","password":"SuperSecretPassword123"}' \
  --region eu-north-1
```

---

## Step 2: Deploy to Kubernetes

```bash
kubectl apply -f 0-namespace.yaml
kubectl apply -f 1-secret-provider-class.yaml
kubectl apply -f 2-service-account.yaml
kubectl apply -f 3-deployment.yaml
```

---

## Step 3: Verification

```bash
# 1. Verify the pod is running:
kubectl get pods -n 12-example

# 2. Check the mounted secret files:
kubectl exec -it deployment/myapp -n 12-example -- ls -la /mnt/secrets

# 3. Check the synced Kubernetes Secret:
kubectl get secret myapp-k8s-secret -n 12-example -o yaml

# 4. Check that environment variables are populated:
kubectl exec -it deployment/myapp -n 12-example -- env | grep MY_
```

Expected output:
```
MY_USERNAME=admin
MY_PASSWORD=SuperSecretPassword123
```
