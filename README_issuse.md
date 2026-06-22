# README — Fix Minikube Service and Ingress 502 Error

## Problem Summary

You faced these issues:

1. `minikube service my-service-k8` failed because Minikube tried to open a browser inside the Ubuntu VM.
2. `curl http://192.168.49.2:30007` returned connection refused.
3. `kubectl port-forward service/my-service-k8 8080:80` returned:

```text
connect(5, AF=2 127.0.0.1:8080): Connection refused
```

4. Ingress returned:

```text
502 Bad Gateway
```

---

## Root Cause

Your Kubernetes configuration had a **port mismatch**.

| Component                    |       Configured Port |           Actual Port |
| ---------------------------- | --------------------: | --------------------: |
| NGINX container              | `containerPort: 8080` | NGINX listens on `80` |
| Service `targetPort`         |                `8080` |        Should be `80` |
| Ingress backend Service port |                  `80` |               Correct |

NGINX logs confirmed that NGINX started successfully, but it listens on port `80`.

Because the Service forwarded traffic to port `8080`, Kubernetes and Ingress could not connect to the NGINX container. This caused `connection refused` and `502 Bad Gateway`.

---

# Correct Architecture

```text
Browser / curl
      |
      v
Ingress Controller
      |
      v
my-service-k8:80
      |
      v
NGINX Pod:80
```

---

# Step 1: Check Pod Status

```bash
kubectl get pods
```

Expected:

```text
NAME                                READY   STATUS    RESTARTS
nginx-deployment-xxxxx-xxxxx        1/1     Running   0
```

Check Pod details:

```bash
kubectl describe pod <pod-name>
```

Example:

```bash
kubectl describe pod nginx-deployment-68c7dff6d-hdbqh
```

Check NGINX logs:

```bash
kubectl logs nginx-deployment-68c7dff6d-hdbqh
```

Expected logs:

```text
nginx/1.x
start worker processes
```

---

# Step 2: Correct Deployment YAML

Create or update `deployment.yaml`.

```bash
nano deployment.yaml
```

Paste:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: sanjeev0181/nginx-app:latest
        ports:
        - containerPort: 80
```

Apply it:

```bash
kubectl apply -f deployment.yaml
```

Check rollout:

```bash
kubectl rollout status deployment/nginx-deployment
```

---

# Step 3: Correct Service YAML

Create or update `service.yaml`.

```bash
nano service.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service-k8
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

Apply:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get svc my-service-k8
```

Expected:

```text
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)
my-service-k8   NodePort   10.x.x.x        <none>        80:30007/TCP
```

---

# Step 4: Verify Service Endpoints

Run:

```bash
kubectl get endpoints my-service-k8
```

Expected:

```text
NAME            ENDPOINTS
my-service-k8   10.244.0.3:80
```

Important:

```text
Correct endpoint: 10.244.0.3:80
Wrong endpoint:   10.244.0.3:8080
```

If the endpoint still shows `8080`, reapply `service.yaml`:

```bash
kubectl apply -f service.yaml
```

---

# Step 5: Test Application Through Service

Run a temporary curl Pod:

```bash
kubectl run test-pod --rm -it \
  --image=curlimages/curl \
  --restart=Never -- \
  curl -I http://my-service-k8:80
```

Expected:

```text
HTTP/1.1 200 OK
Server: nginx
```

This confirms:

```text
Pod → Service → NGINX application
```

is working.

---

# Step 6: Enable NGINX Ingress

```bash
minikube addons enable ingress
```

Check controller Pods:

```bash
kubectl get pods -n ingress-nginx
```

Expected:

```text
ingress-nginx-controller-xxxxx   1/1   Running
```

---

# Step 7: Create Ingress YAML

Create `ingress.yaml`:

```bash
nano ingress.yaml
```

Paste:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: mydomain.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service-k8
            port:
              number: 80
```

Apply:

```bash
kubectl apply -f ingress.yaml
```

Verify:

```bash
kubectl get ingress
kubectl describe ingress my-app-ingress
```

Expected backend:

```text
my-service-k8:80
```

---

# Step 8: Get Minikube IP

```bash
minikube ip
```

Example output:

```text
192.168.49.2
```

Check Ingress Controller NodePort:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

Example:

```text
80:31829/TCP
443:30149/TCP
```

Here:

```text
HTTP Ingress URL = http://192.168.49.2:31829
```

---

# Step 9: Test Ingress

Run:

```bash
curl -H "Host: mydomain.local" http://192.168.49.2:31829
```

Expected result:

```html
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
</body>
</html>
```

---

# Step 10: If You Get `502 Bad Gateway`

Run these commands:

```bash
kubectl get pods
kubectl get svc my-service-k8
kubectl get endpoints my-service-k8
kubectl describe ingress my-app-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50
```

The endpoint must be:

```text
my-service-k8   <pod-ip>:80
```

If it shows:

```text
<pod-ip>:8080
```

your Service is incorrect.

Apply the corrected Service again:

```bash
kubectl apply -f service.yaml
```

Restart Ingress Controller:

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

Wait:

```bash
kubectl get pods -n ingress-nginx -w
```

Test again:

```bash
curl -H "Host: mydomain.local" http://192.168.49.2:31829
```

---

# Step 11: Access From Windows Browser

Your Minikube cluster is inside Ubuntu VirtualBox. Use port-forward for the Ingress Controller:

```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8081:80
```

Keep this command running.

Open Windows hosts file as Administrator:

```text
C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
127.0.0.1 mydomain.local
```

Open in Windows browser:

```text
http://mydomain.local:8081
```

---

# Useful Commands

```bash
kubectl get all
kubectl get pods -o wide
kubectl get svc
kubectl get ingress
kubectl get endpoints
kubectl describe pod <pod-name>
kubectl describe svc my-service-k8
kubectl describe ingress my-app-ingress
kubectl logs <pod-name>
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50
minikube ip
minikube status
```

---

# Key Learning

| Kubernetes Object  | Purpose                                                     |
| ------------------ | ----------------------------------------------------------- |
| Deployment         | Creates and manages NGINX Pods                              |
| Pod                | Runs the NGINX container                                    |
| Service            | Connects traffic to Pods using labels                       |
| `targetPort`       | Must match the actual application port inside the container |
| Ingress            | Routes HTTP traffic to Kubernetes Services                  |
| Ingress Controller | Reads Ingress rules and forwards traffic to Services        |
| NodePort           | Opens a port on the Minikube node                           |
| Port-forward       | Creates local temporary access to a Service or Pod          |
