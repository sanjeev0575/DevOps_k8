For Ubuntu **26.04**, use the **Docker driver**. It is the simplest Minikube setup and does not require VirtualBox or nested virtualization. Minikube requires at least 2 CPUs, 2 GB free RAM, and 20 GB disk space. ([minikube][1])

## Step 1: Update Ubuntu packages

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Step 2: Install required packages

```bash
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg conntrack
```

---

## Step 3: Install Docker

```bash
sudo apt install -y docker.io
```

Start and enable Docker:

```bash
sudo systemctl enable --now docker
```

Add your current user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Apply the group change:

```bash
newgrp docker
```

Verify Docker:

```bash
docker --version
docker ps
```

If `docker ps` works without `sudo`, Docker is configured correctly.

---

## Step 4: Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
```

Verify:

```bash
kubectl version --client
```

The Kubernetes project publishes the current stable `kubectl` binary through its official release URL. ([Kubernetes][2])

---

## Step 5: Install Minikube

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
```

```bash
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

```bash
rm minikube-linux-amd64
```

Verify:

```bash
minikube version
```

This is the current official Linux AMD64 installation method. ([minikube][1])

---

## Step 6: Start Minikube using Docker driver

```bash
minikube start --driver=docker
```

Make Docker the default Minikube driver:

```bash
minikube config set driver docker
```

Check status:

```bash
minikube status
```

Expected output:

```text
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Check Kubernetes node:

```bash
kubectl get nodes
```

Expected:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...   ...
```

The Docker driver is supported on Linux and Minikube documents `minikube start --driver=docker` as the standard command. ([minikube][3])

---

## Step 7: Enable Ingress addon

```bash
minikube addons enable ingress
```

Verify it:

```bash
kubectl get pods -n ingress-nginx
```

Wait until the Ingress controller Pod is `Running`.

---

## Important commands

### Stop Minikube

```bash
minikube stop
```

### Start again

```bash
minikube start
```

### Check cluster information

```bash
kubectl cluster-info
```

### Open Minikube dashboard

```bash
minikube dashboard
```

### Delete Minikube cluster

```bash
minikube delete
```

`minikube delete` removes the local cluster and its associated files. ([minikube][4])

[1]: https://minikube.sigs.k8s.io/docs/start/?utm_source=chatgpt.com "minikube start - Kubernetes"
[2]: https://kubernetes.io/docs/tasks/tools/?utm_source=chatgpt.com "Install Tools"
[3]: https://minikube.sigs.k8s.io/docs/drivers/docker/?utm_source=chatgpt.com "docker | minikube - Kubernetes"
[4]: https://minikube.sigs.k8s.io/docs/commands/delete/?utm_source=chatgpt.com "delete - Minikube - Kubernetes"
