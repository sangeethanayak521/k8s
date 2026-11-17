# Lesson 196: Example 5 — AWS Load Balancer Controller (NLB in IP Mode)

This directory contains the Kubernetes manifests from Anton Putra's tutorial:
`https://github.com/antonputra/tutorials/tree/main/lessons/196/5-example`

It demonstrates exposing a Kubernetes application directly through an **AWS Network Load Balancer (NLB)** in **IP Target Mode** managed by the **AWS Load Balancer Controller**.

---

## 📁 File Structure

* `original/`: Exact, unmodified files fetched directly from Anton Putra's GitHub repository.
* `0-namespace.yaml`: Creates the dedicated `5-example` namespace.
* `1-deployment.yaml`: **Corrected** Deployment with high availability replicas and health probes.
* `2-service.yaml`: **Corrected** Service with HTTP/80 mapping and active HTTP health checking.

---

## 🔍 Mistakes & Shortcomings in the Original Code

| Component | Original Code | Issue / Mistake | Correction Made |
| :--- | :--- | :--- | :--- |
| **Service Port** | `port: 8080` | Browsers default to port 80. Accessing `http://<NLB-DNS>` results in `ERR_CONNECTION_REFUSED` unless the user explicitly appends `:8080`. | Added standard port `80` (mapping to container `8080`), while preserving `8080` for backwards compatibility. |
| **NLB Health Check** | No health check annotations | Default NLB target health check performs a raw **TCP handshake**. It does not verify whether the Go Fiber application is actually healthy or returning HTTP 500 errors. | Configured HTTP health check on path `/health` on port `8080`. |
| **Pod Readiness Probe** | None | When using `target-type: ip`, the AWS Load Balancer Controller registers the pod IP as soon as Kubernetes considers it ready. Without a readiness probe, traffic is sent before the web server is listening. | Added `readinessProbe` targeting `/health` with a 3-second initial delay. |
| **Pod Liveness Probe** | None | If the container deadlocks or hangs, Kubernetes will never restart it automatically. | Added `livenessProbe` on `/health`. |
| **High Availability** | `replicas: 1` | Single point of failure in a Multi-AZ cluster. | Increased to `replicas: 2`. |
| **Port Protocol** | Missing `protocol: TCP` | Implicit rather than explicit. | Explicitly declared `protocol: TCP`. |

---

## 🚀 How to Deploy & Verify

### 1. Apply Manifests
```bash
kubectl apply -f 5-example/0-namespace.yaml
kubectl apply -f 5-example/1-deployment.yaml
kubectl apply -f 5-example/2-service.yaml
```

### 2. Check Pods and Service
```bash
kubectl get pods -n 5-example -o wide
kubectl get svc -n 5-example
```

### 3. Test Ingress Traffic
Get the external hostname of the NLB:
```bash
NLB_HOST=$(kubectl get svc myapp -n 5-example -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo $NLB_HOST
```

Test standard HTTP (port 80):
```bash
curl http://$NLB_HOST/health
# Output: {"status":"ok"}

curl http://$NLB_HOST/about
# Output: {"service":"myapp","version:":"v0.1.4"}
```
