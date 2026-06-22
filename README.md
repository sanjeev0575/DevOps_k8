# DevOps_k8

Create a file named **`README.md`** in your GitHub repository and paste this content:

# Nginx Application Deployment on Kubernetes

This project deploys a simple Nginx web application to Kubernetes using:

* Docker
* Docker Hub
* Kubernetes Deployment
* Kubernetes Service
* Kubernetes Ingress
* Minikube

## Project Structure

```text
nginx-k8s-project/
│
├── index.html
├── Dockerfile
|-- ingress.yaml
├── nginx-app.yaml
└── README.md
```

## Prerequisites

Install the following tools before starting:

* Docker
* Minikube
* kubectl
* Git
* Docker Hub account

Check installed versions:

```bash
docker --version
minikube version
kubectl version --client
git --version
```

---

# Step 1: Create `index.html`

Create an `index.html` file.

```html
<!DOCTYPE html>
<html>
<head>
  <title>Nginx Kubernetes Application</title>
</head>
<body>
  <h1>Welcome to Nginx Application</h1>
  <h2>Application deployed successfully using Kubernetes Ingress</h2>
</body>
</html>
```

---

# Step 2: Create Dockerfile

Create a file named `Dockerfile`.

```dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## Dockerfile Explanation

| Line                | Description                                   |
| ------------------- | --------------------------------------------- |
| `FROM nginx:alpine` | Uses lightweight Nginx base image             |
| `COPY index.html`   | Copies application page into Nginx web folder |
| `EXPOSE 80`         | Nginx listens on port 80                      |
| `CMD`               | Starts the Nginx server                       |

---

# Step 3: Build Docker Image

Run this command from the project directory:

```bash
docker build -t sanjeev0181/nginx-app:latest .
```

Check the image:

```bash
docker images
```

Expected image name:

```text
sanjeev0181/nginx-app
```

---

# Step 4: Test Docker Container Locally

Run the Docker container:

```bash
docker run -d -p 8080:80 --name nginx-container sanjeev0181/nginx-app:latest
```

Open in browser:

```text
http://localhost:8080
```

Stop and remove the test container:

```bash
docker stop nginx-container
docker rm nginx-container
```

---

# Step 5: Push Image to Docker Hub

Login to Docker Hub:

```bash
docker login
```

Push the image:

```bash
docker push sanjeev0181/nginx-app:latest
```

Verify that the image is available in your Docker Hub repository.

---

# Step 6: Start Minikube

Start Minikube:

```bash
minikube start
```

Check cluster status:

```bash
minikube status
```

Check Kubernetes nodes:

```bash
kubectl get nodes
```

Expected status:

```text
Ready
```

---

# Step 7: Enable Nginx Ingress Controller

Enable the Ingress addon:

```bash
minikube addons enable ingress
```

Verify the Ingress controller Pod:

```bash
kubectl get pods -n ingress-nginx
```

Wait until the Pod status becomes:

```text
Running
```

---

# Step 8: Create Kubernetes YAML File

Create a file named `nginx-app.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: default
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
          imagePullPolicy: Always
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-service-k8
  namespace: default
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: public-app-ingress
  namespace: default
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

---

# Step 9: Deploy Application to Kubernetes

Apply the YAML file:

```bash
kubectl apply -f nginx-app.yaml
```

Check Deployment:

```bash
kubectl get deployments
```

Check Pods:

```bash
kubectl get pods
```

Check Service:

```bash
kubectl get svc
```

Check Ingress:

```bash
kubectl get ingress
```

---

# Step 10: Verify Service Endpoints

Run:

```bash
kubectl get endpoints my-service-k8
```

The command should show the Pod IP and port `80`.

If no endpoint is displayed, verify that the Service selector and Pod labels are the same:

```yaml
app: nginx
```

---

# Step 11: Get Minikube IP

Run:

```bash
minikube ip
```

Example output:

```text
192.168.49.2
```

---

# Step 12: Configure Local Domain

Open the Ubuntu hosts file as Administrator:

```text
vi /etc/hosts
```

Add the following line:

```text
192.168.49.2 mydomain.local
```

Replace `192.168.49.2` with the output of your `minikube ip` command.

---

# Step 13: Access Application

Open this URL in your browser:

```text
http://mydomain.local
```

You should see the Nginx application page.

---

# Application Flow

```text
GitHub Repository
        |
        v
Dockerfile + index.html
        |
        v
Docker Image Build
        |
        v
Docker Hub Repository
        |
        v
Kubernetes Deployment
        |
        v
Nginx Pod
        |
        v
ClusterIP Service
        |
        v
Nginx Ingress Controller
        |
        v
http://mydomain.local
```

---

# Important Commands

## Check all resources

```bash
kubectl get all
```

## View Deployment details

```bash
kubectl describe deployment nginx-deployment
```

## View Pod logs

```bash
kubectl logs -l app=nginx
```

## View Service details

```bash
kubectl describe svc my-service-k8
```

## View Ingress details

```bash
kubectl describe ingress public-app-ingress
```

## Delete application

```bash
kubectl delete -f nginx-app.yaml
```

---

# Troubleshooting

## Pod is not running

Check Pod status:

```bash
kubectl get pods
```

Check Pod events:

```bash
kubectl describe pod <pod-name>
```

## ImagePullBackOff error

Verify that the Docker image exists in Docker Hub:

```bash
docker pull sanjeev0181/nginx-app:latest
```

## Ingress is not working

Check Ingress controller:

```bash
kubectl get pods -n ingress-nginx
```

Check Ingress configuration:

```bash
kubectl describe ingress public-app-ingress
```

## Service has no endpoints

Check labels:

```bash
kubectl get pods --show-labels
```

The Deployment labels and Service selector must both use:

```yaml
app: nginx
```

---

# Note

`mydomain.local` is used only for local Minikube testing.

For real public internet access, use:

* A registered domain name
* Public DNS record
* Cloud Load Balancer or public Ingress Controller
* HTTPS certificate

You can commit and push it to GitHub:

```bash
git add README.md
git commit -m "Add Kubernetes deployment documentation"
git push origin main
```
