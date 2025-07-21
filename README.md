# EKS Ingress Controller Architecture with Route 53 DR Failover

## 📌 Objective

To implement a scalable, secure, and DR-ready ingress architecture on AWS EKS that:
- Uses **a single external-facing ALB** per cluster.
- Applies **IP allowlisting per application** using the **NGINX ingress controller**.
- Leverages **AWS Route 53 failover** DNS strategy across two clusters (Primary + DR).
- Optionally integrates **AWS WAF** for application-layer protection at the ALB level.

---

## 📂 Components

| Component | Description |
|----------|-------------|
| **AWS ALB Ingress Controller** | Handles all external ingress; publicly exposed; integrates with Route 53 and WAF. |
| **NGINX Ingress Controller** | Internal ingress controller; applies per-app rules including IP allowlisting. |
| **Route 53 DNS Failover** | Provides single DNS endpoint with health-based failover between regions. |

---

## 🏗️ Architecture

```

Internet
|
\[ Route53 DNS (Failover) ]
|
\[ Public ALB via AWS ALB Ingress Controller ]
|
\[ NGINX Ingress Controller Service (NodePort/IP) ]
|
\[ Per-App Ingress Rules with IP Whitelist ]
|
\[ App Services ]

````

---

## 🛠️ Prerequisites

- Two EKS clusters (Primary and DR).
- IAM roles and OIDC provider set up.
- External-dns (optional, for auto DNS management).
- Helm installed.

---

## 🚀 Step-by-Step Implementation

### 1. Deploy AWS ALB Ingress Controller

```bash
helm repo add eks https://aws.github.io/eks-charts
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<YOUR_CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1
````

Ensure your service account is created with the correct IAM policies.

---

### 2. Deploy NGINX Ingress Controller (Internal)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.ingressClassResource.name=nginx \
  --set controller.ingressClass=nginx \
  --set controller.scope.enabled=true \
  --set controller.scope.namespace=ingress-nginx
```

---

### 3. ALB Ingress to Route Traffic to NGINX

```yaml
# alb-nginx-forwarder.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: alb-to-nginx
  namespace: ingress-nginx
  annotations:
    alb.ingress.kubernetes.io/group.name: external-alb-group
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: <ACM_ARN>
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/subnets: subnet-xxx,subnet-yyy,subnet-zzz
    alb.ingress.kubernetes.io/wafv2-acl-arn: <optional-waf-arn>
spec:
  ingressClassName: alb
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /*
        pathType: ImplementationSpecific
        backend:
          service:
            name: nginx-ingress-controller
            port:
              number: 80
```

---

### 4. NGINX Ingress with IP Allowlisting

```yaml
# nginx-app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/whitelist-source-range: "122.169.70.122/32,165.225.122.0/23"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /myapp
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
```

---

### 5. Route 53 DNS Configuration

* Domain: `app.example.com`
* Create two DNS records (type A or CNAME):

  * **Primary record**: ALB DNS of Primary EKS cluster — set as **"Primary"**
  * **Failover record**: ALB DNS of DR EKS cluster — set as **"Secondary"**
* Attach **Route 53 health checks** to the ALB `/healthz` endpoints.

---

## ✅ Benefits

* 🔒 Secure app-level access control (IP whitelist) using NGINX.
* 🌐 Single ALB DNS per cluster reduces Route 53 record sprawl.
* 🔁 Seamless failover via Route 53.
* 🛡️ AWS WAF support on the ALB.
* 🔧 Easier app-to-app routing with NGINX.

---

## 📎 Notes

* Ensure NGINX controller has correct RBAC and network access.
* You can further enhance this with `external-dns` to auto-manage Route 53 entries.
* Set up `/healthz` endpoints per app for failover health checks.

---

## 🧪 Health Check Example (Optional)

Deploy a dummy `/healthz` endpoint using a simple pod or sidecar in NGINX namespace:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthz
  namespace: ingress-nginx
spec:
  containers:
  - name: healthz
    image: hashicorp/http-echo
    args:
      - "-text=ok"
    ports:
    - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: healthz
  namespace: ingress-nginx
spec:
  selector:
    app: healthz
  ports:
  - port: 80
    targetPort: 5678
```

---

## 👨‍💻 Author

DevOps Team — Contact: Mani V

```

---
