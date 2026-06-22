# End-to-End CI/CD Pipeline: Flask Microservice on Custom Kubernetes

## Project Overview

This project demonstrates a fully automated CI/CD pipeline deploying a Python Flask web application. Unlike managed cloud solutions (like AWS EKS), the Kubernetes cluster in this project was built entirely from scratch using `kubeadm` on standalone virtual machines (AWS EC2 instances). 

Whenever new code is pushed to this GitHub repository, Jenkins automatically builds a new Docker image, pushes it to Docker Hub, and rolls out the update to the live Kubernetes cluster. Finally, traffic is routed to the application via an Application Load Balancer (ALB) for security and load distribution.

---

## Architecture & Traffic Flow

1. **Developer Push:** Code is pushed to the `main` branch of this GitHub repository.
2. **Webhook Trigger:** GitHub sends a webhook payload to the Jenkins server.
3. **CI/CD Pipeline (Jenkins):**
   * Pulls the latest code using a GitHub Personal Access Token (PAT).
   * Builds a lightweight Docker image using `python:3.11-slim` and `Gunicorn`.
   * Authenticates and pushes the tagged image to Docker Hub.
   * Uses a securely stored `kubeconfig` to execute `kubectl apply` commands against the Kubernetes Master Node.
4. **Kubernetes Orchestration:** The cluster pulls the new image and updates the Pods with zero downtime.
5. **Traffic Routing:**
   * User requests hit the **Application Load Balancer (ALB)** on Port `80`.
   * The ALB forwards traffic to the Kubernetes Worker Nodes on the **NodePort** (e.g., `3000X`).
   * `kube-proxy` routes the traffic internally to the Pods running Gunicorn on Port `5000` (the Flask app exposed port).

---

## Technology Stack

* **Application:** Python 3.11, Flask, Gunicorn, HTML Templates
* **Containerization:** Docker, Docker Hub
* **CI/CD:** Jenkins, GitHub Webhooks
* **Orchestration:** Kubernetes (kubeadm, kubectl, manual Master/Worker nodes)
* **Cloud Infrastructure:** AWS Virtual Machines (EC2), Application Load Balancer (ALB), Security Groups

---

## Step-by-Step Implementation Guide

### 1. The Application & Dockerization
The application is a lightweight web server using the Flask framework to serve an `index.html` frontend template. It utilizes Gunicorn as the production WSGI HTTP server. The application image is built using Docker and stored in Docker Hub.

* **Base Image:** Uses `python:3.11-slim` for a secure, stable, and minimal footprint environment.
* **Execution:** The container runs via:
  `CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]`

**Architecture Flow Diagram**
![flow diagram](resources/image-flow.png)

### 2. Manual Kubernetes Cluster Setup
A multi-node Kubernetes cluster was provisioned manually within the AWS cloud. This approach was chosen to gain deeper infrastructure understanding and reduce the costs associated with managed services like EKS.

*Note: Every instance is accessible using its own public IP.*

**Cluster Configuration Steps:**
1. Provision two EC2 instances with a minimum of 2 CPU cores and 4 GB of RAM. Name them `master` and `worker`.
2. Install Docker on both nodes following the official documentation.
3. Install `kubeadm`, `kubelet`, and `kubectl` using the provided configuration <a href="/resources/">scripts</a>.
4. Run <a href="/resources/common.sh">common.sh</a> on both instances. This script handles prerequisites like disabling swap, loading kernel modules, and setting sysctl parameters.
5. Run <a href="/resources/master.sh">master.sh</a> exclusively on the Master node to initialize the control plane.
6. Upon successful initialization, the Master node will output a join command. Copy and run this command on the Worker node with `sudo` privileges to attach it to the cluster.
   ```bash
   # Example output:
   kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --cri-socket unix:///var/run/containerd/containerd.sock
   ```
7. Verify the cluster status by running `kubectl get nodes` on the Master node. Both nodes should appear in a `Ready` state.
8. Apply necessary socket permissions: `sudo chmod 666 /var/run/docker.sock`

### 3. Jenkins Server Configuration
Jenkins handles the CI/CD automation. The pipeline script is hosted directly within the Jenkins project configuration for streamlined execution.

**Jenkins Infrastructure Setup:**
1. Provision an EC2 instance with a minimum of 2 CPU cores and 2 GB of RAM. Name it `jenkins`.
2. Install <a href="[https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)">Docker</a> and <a href="[https://kubernetes.io/docs/tasks/tools/kubectl](https://kubernetes.io/docs/tasks/tools/kubectl)">kubectl</a>.
3. Install Jenkins following the <a href="[https://www.jenkins.io/doc/book/installing/](https://www.jenkins.io/doc/book/installing/)">official documentation</a>.
4. Grant Jenkins permissions to interact with the Docker daemon:
   `sudo chmod 666 /var/run/docker.sock`

**AWS EC2 Instances List**
![instances to get](<resources/Screenshot 2026-04-08 at 9.05.07 PM.png>)

**Key Jenkins Configurations:**
* **Plugins:** Installed Docker Pipeline, Kubernetes CLI, and GitHub Integration.
* **Permissions:** Added the `jenkins` user to the `docker` Linux group to allow seamless image building.
* **Tooling:** Installed the `kubectl` binary directly on the Jenkins server to enable remote cluster management.

**Secured Credentials:**
* `github-creds`: Personal Access Token (PAT) for checking out source code.
* `dockerhub-creds`: Authentication for pushing built images to the registry.
* `k8s-kubeconfig`: The Master node's `kubeconfig` file, stored securely as a Secret File in Jenkins.

### **Pipeline Configuration**
The complete pipeline code can be found here: <a href="/resources/Jenkinsfile">Jenkinsfile</a>

### 4. Kubernetes Manifests
The deployment logic is maintained in the `k8s/` directory of this repository.

* **`deployment.yaml`:** Manages the Pod replicas and defines the container image. The Jenkins pipeline dynamically injects the unique `build-${BUILD_NUMBER}` tag into this file before applying it to the cluster.
* **`service.yaml`:** Exposes the application Pods internally and externally using a `NodePort`.

**Pod and Service Status Verification** \
![pod_svc](resources/podsandsvc.jpeg)

### 5. Automation via Webhooks
A webhook is configured within the GitHub repository settings, pointing to:
`http://<JENKINS_IP>:8080/github-webhook/` (Content Type: `application/json`).

The Jenkins project is configured with the **"GitHub hook trigger for GITScm polling"** option enabled, achieving full Continuous Deployment (CD) upon every code push.

### 6. Application Load Balancer (ALB) Setup
To provide a secure, single entry point for end-users:

* Created an ALB Target Group pointing to the Worker Node IPs on the assigned `NodePort`.
* Configured Security Groups:
  * **ALB Security Group:** Allows inbound HTTP traffic (Port 80) from Anywhere (`0.0.0.0/0`).
  * **Worker Node Security Group:** Allows inbound Custom TCP traffic on the `NodePort` **only** from the ALB Security Group, preventing direct external access to the nodes.

### **Final Output**
![output webpage](<resources/Screenshot 2026-04-09 at 10.28.41 AM.png>)


### 7. Setup for monitoring
To monitor the full kubernetes cluster:

1. Install helm (The Kubernetes Package manager) using the bash script.
2. Deploying Prometheus using helm repo, add and install within monitoring namespace.

```
# Add Prometheous official repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
helm repo update

# Create the dedicated monitoring namespace
kubectl create namespace monitoring

# Install Prometheus with local ephemeral storage

helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --set server.persistentVolume.enabled=false \
  --set alertmanager.persistentVolume.enabled=false

# Check Status
kubectl get pods -n monitoring -w
```

3. Grafana setup for proper visualization, using helm, By using Helm delpoying grafana inside monitoring namespace

```
# Add the official Grafana repository
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Grafana
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=false

# Check Status
kubectl get pods -n monitoring -w
```

4. Setting up Grafana's password, dashboards etc.
```
# Retrieve the Auto-Generated Admin Password, copy the resulted password. 
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
5. Access the Dashboard by exposing 3000 port in private network and enter username as ```admin``` and ```paste copied password```.

6. Setting up Dashboard inside Grafana.
   - Click the Dashboards icon (four squares) on the left menu.
   - Click the New button -> Import.
   - In the "Import via grafana.com" box, type the ID number: 1860
   - (Examples of Dashboard can be loaded for monitoring: ID number 1860: Old best dashboard, ID number 315: Full Cluster monitoring, ID number: 6336: New proper dashboard for monitoring.. etc..)
   - Click Load.
   - At the bottom of the next screen, select your Prometheus data source from the dropdown.
   - Click Import.


---

## Challenges Overcome

* **Docker Socket Permissions:** Resolved `permission denied while trying to connect to the docker API` by correctly managing Linux user groups (`sudo usermod -aG docker jenkins`).
* **Remote Cluster Execution:** Resolved `kubectl: not found` exit code 127 in the Jenkins pipeline by manually installing the Kubernetes CLI binaries directly onto the Jenkins host server.
* **Gunicorn Entrypoints:** Refined the `Dockerfile` syntax to properly execute Gunicorn in a containerized environment without array format syntax errors.