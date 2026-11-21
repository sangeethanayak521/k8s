# 9-example: Cert-Manager Automated Let's Encrypt SSL/TLS

This example demonstrates zero-touch SSL/TLS certificate management using **Jetstack Cert-Manager** and **Let's Encrypt** ACME HTTP-01 challenges.

## Prerequisites
1. **Cert-Manager installed:** Deployed via `17-cert-manager.tf`.
2. **Ingress NGINX installed:** Deployed via `16-nginx-ingress.tf` with class `external-nginx`.
3. **Public DNS Record:** In your DNS management console for `aiguru.website`, create a CNAME record:
   - **Host:** `ex9` (or `ex9.aiguru.website`)
   - **Value:** The External DNS name of your NGINX Load Balancer:
     ```bash
     kubectl get svc -n ingress external-ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
     ```

## Deployment Steps

```bash
# 1. Apply ClusterIssuer (production ACME endpoint)
kubectl apply -f 0-cluster-issuer.yaml

# 2. Apply Namespace, Deployment, Service, and Ingress
kubectl apply -f 1-namespace.yaml
kubectl apply -f 2-deployment.yaml
kubectl apply -f 3-service.yaml
kubectl apply -f 4-ingress.yaml
```

## How It Works
1. Ingress metadata specifies `cert-manager.io/cluster-issuer: http-01-production`.
2. Cert-Manager detects the Ingress and automatically creates a `Certificate` and `Order` resource.
3. An ACME HTTP-01 challenge pod and temporary ingress rule (`/.well-known/acme-challenge/...`) are spawned.
4. Let's Encrypt servers send an HTTP request to `http://ex9.aiguru.website/.well-known/acme-challenge/...`.
5. Once validated, Let's Encrypt signs the certificate and Cert-Manager saves the TLS certificate and private key in the secret `ex9-aiguru-website-tls`.
6. NGINX Ingress reloads its SSL configuration, enabling valid HTTPS at `https://ex9.aiguru.website`.

## Verification Commands
```bash
# Check certificate issuance status (READY should become True):
kubectl get certificate -n 9-example

# Inspect certificate request details:
kubectl describe certificate ex9-aiguru-website-tls -n 9-example

# Test secure HTTPS endpoint:
curl -v https://ex9.aiguru.website
```
