# 🚀 Google Kubernetes Engine (GKE) 

> **Practical guide to GKE**: covering cluster creation (Standard & Autopilot), node pools, deployments, Services, autoscaling (HPA, Cluster Autoscaler), rolling updates, and GKE features. 

---

## 📌 Project Overview

This README demonstrates end-to-end GKE usage, including: 

- **Cluster Setup:** Creating Standard and Autopilot clusters with `gcloud` (zonal/regional).  
- **Node Pools:** Adding/managing node pools and enabling autoscaling.  
- **Deployments:** Running containers via imperative commands and declarative YAML.  
- **Networking:** Exposing apps with Services (`ClusterIP`/`NodePort`/`LoadBalancer`) and Ingress.  
- **Scaling:** Horizontal Pod Autoscaler (HPA) for pods and Cluster Autoscaler for nodes【7†L991-L999】【20†L784-L793】.  
- **Updates:** Rolling updates and rollbacks of applications.  
- **Built-in Services:** Automatic node upgrades/repairs【26†L765-L773】【30†L1-L4】 and GCP integrations (Logging/Monitoring, Load Balancer).  

---

## ❓ Why Google Kubernetes Engine (GKE)?

GKE is Google Cloud’s managed Kubernetes service, offering:

- **Managed Control Plane:** Google runs the API server, etcd, scheduler, and auto-upgrades the control plane.  
- **Autopilot vs Standard:** In Autopilot mode, GKE also manages node provisioning and autoscaling. Standard mode gives you full control over node pools.  
- **Scalability:** Supports multi-zone clusters with auto-scaling (pods and nodes) and release channels.  
- **Integrated Services:** Built-in HTTP(S) Load Balancing (via Service LoadBalancer or Ingress) and Cloud Monitoring/Logging out-of-the-box.  
- **Productivity:** Simplifies operations (auto-repair, auto-upgrade of nodes), so teams can focus on applications, not infrastructure.  

---

## 🏗️ GKE Architecture (Standard vs Autopilot)

- **Control Plane:** Fully managed by Google. Runs `kube-apiserver`, scheduler, controllers, and etcd (cluster state).  
- **Nodes:** Your worker VMs. In **Standard** mode you provision and manage node pools; in **Autopilot**, Google manages nodes/auto-scaling for you.  
- **Networking:** GKE clusters connect to VPC networks. Services of type **LoadBalancer** automatically create a Google Cloud external load balancer. Ingress resources use the built-in GKE Ingress controller to provision HTTP(S) load balancers.  
- **Storage & Config:** GKE integrates with Cloud Storage, Filestore, and ConfigMaps/Secrets for configuration and persistent data.  

---

## 🛠️ Tools Used

| Tool                  | Purpose                                          |
|-----------------------|--------------------------------------------------|
| **gcloud CLI**        | Create/manage GKE clusters and node pools        |
| **kubectl**           | Kubernetes CLI for deploying and managing apps   |
| **Google Cloud Console** | Web UI for monitoring and cluster operations  |
| **Docker / Container Registry** | Build and store container images          |
| **Horizontal Pod Autoscaler** | Automatically scales pods based on metrics |


---

## 🧰 Repository Structure

```bash
.
├── deployment.yaml         # Example Deployment + Service YAML
├── hpa.yaml                # (Optional) HPA configuration
├── README.md               # This file
└── scripts/
    ├── create-cluster.sh   # (Optional) Automation scripts
    ├── deploy-app.sh
    └── cleanup.sh
```

---

## 🚀 1️⃣ Set Up GKE Cluster

1. **Configure gcloud:** Install the Google Cloud SDK and run `gcloud init`. Choose your project and set the default zone/region (`gcloud config set compute/zone us-central1-a` etc.).  

2. **Standard Cluster (Zonal):**
   ```bash
   gcloud container clusters create my-cluster \
     --zone us-central1-a \
     --num-nodes 3 \
     --machine-type e2-medium \
     --release-channel regular
   ```
   > *By default, new Standard clusters have **Node Auto-Upgrade** and **Auto-Repair** enabled.*

3. **Autopilot Cluster (Regional):**
   ```bash
   gcloud container clusters create-auto my-auto-cluster \
     --region us-central1
   ```
   Autopilot clusters automatically manage node pools and scaling, making them effectively “serverless” Kubernetes.  

4. **Get Credentials:** Configure `kubectl` to use your new cluster:
   ```bash
   gcloud container clusters get-credentials my-cluster --zone us-central1-a
   ```
   This sets the context so `kubectl` commands target your cluster.

5. **Verify Nodes:**  
   ```bash
   kubectl get nodes
   ```
   Expect to see 3 READY nodes (for the 3 specified) in Standard mode.

   *Add a screenshot of the GKE cluster and nodes in the Google Cloud Console here.*

---

## 🚀 2️⃣ Manage Node Pools (Standard Mode)

- **View Node Pools:**  
  ```bash
  gcloud container node-pools list --cluster my-cluster --zone us-central1-a
  ```

- **Add a Node Pool:** (e.g., for GPU or different sizing)  
  ```bash
  gcloud container node-pools create gpu-pool \
    --cluster my-cluster --zone us-central1-a \
    --machine-type n1-standard-4 --num-nodes 2 \
    --accelerator type=nvidia-tesla-k80,count=1
  ```
  This creates a separate node pool named `gpu-pool`.

- **Enable Autoscaling on a Node Pool:**  
  ```bash
  gcloud container node-pools update default-pool \
    --cluster my-cluster --zone us-central1-a \
    --enable-autoscaling --min-nodes=1 --max-nodes=5
  ```
  This lets GKE auto-scale the pool between 1 and 5 nodes based on workload. (Autopilot has built-in autoscaling by design.)  

*Add a screenshot of the Node Pools page in the console here.*

---

## 🚀 3️⃣ Deploy an Application

1. **Imperative Pod/Deployment:**  
   ```bash
   kubectl create deployment nginx-deploy --image=nginx:1.21
   kubectl get deployments,pods
   ```
   This creates a Deployment `nginx-deploy` running the `nginx:1.21` image.

2. **Expose Pod via Service:**  
   ```bash
   kubectl expose deployment nginx-deploy \
     --type=NodePort --port=80
   kubectl get svc nginx-deploy
   ```
   - `NodePort` allocates a port on each node. On GKE, often we prefer `LoadBalancer` to get an external IP.

3. **Declarative Deployment (YAML):**  
   Create a file `deployment.yaml`: 
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deploy
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels: {app: nginx}
       spec:
         containers:
         - name: nginx
           image: nginx:1.21
           ports: [{containerPort: 80}]
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-svc
   spec:
     selector: {app: nginx}
     ports: [{port: 80, targetPort: 80}]
     type: LoadBalancer
   ```
   Apply it:
   ```bash
   kubectl apply -f deployment.yaml
   kubectl get pods,svc
   ```
   The Service of type `LoadBalancer` will create a Google Cloud external LB and assign an external IP.

4. **Check Deployment:**  
   ```bash
   kubectl get deployments
   kubectl get pods
   kubectl get svc
   ```
   Wait for the EXTERNAL-IP to appear under the LoadBalancer service.

*Add a screenshot of `kubectl get pods` and `kubectl get svc` (showing the External IP) here.*

---

## 🚀 4️⃣ Access the Application

- **Via External IP:** Once the LoadBalancer IP is ready, you can curl or browse it:
  ```bash
  curl http://<EXTERNAL_IP>
  ```
- **Port Forwarding:** Alternatively:
  ```bash
  kubectl port-forward svc/nginx-svc 8080:80
  ```
  Then open [http://localhost:8080](http://localhost:8080) to see the NGINX welcome page.

- **Ingress (HTTP(S) Load Balancer):** For advanced L7 routing, create an `Ingress` resource. GKE’s Ingress controller will provision an HTTP(S) LB automatically.

---

## 🚀 5️⃣ Scaling & Autoscaling

- **Manual Scale:**  
  ```bash
  kubectl scale deployment/nginx-deploy --replicas=3
  kubectl get pods
  ```
  This changes the replica count explicitly.

- **Horizontal Pod Autoscaler (HPA):** Automatically scale pods based on metrics (CPU by default).  
  ```bash
  kubectl autoscale deployment nginx-deploy --cpu-percent=50 --min=1 --max=5
  ```
  This creates an HPA that will keep CPU ~50%, scaling between 1–5 replicas.  
  ```bash
  kubectl get hpa
  ```
  Check HPA status. 

  *Add screenshot of `kubectl get hpa` showing current replicas.*

- **Cluster Autoscaler:** GKE can auto-scale node pool sizes within defined bounds.  
  You can create a cluster with autoscaling flags, e.g.:  
  ```bash
  gcloud container clusters create my-autoscale-cluster \
    --zone us-central1-a --num-nodes=3 \
    --enable-autoscaling --min-nodes=1 --max-nodes=5
  ```  
  Or enable on an existing node pool as shown above. The autoscaler will **add/remove VM nodes** based on pending pods.  
  *Add screenshot of node count changing in Cloud Console (or `kubectl get nodes`).*

---

## 🚀 6️⃣ Rolling Updates

Use `kubectl rollout` to update application images without downtime:

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.22
kubectl rollout status deployment/nginx-deploy
```

This updates pods one by one. To view history:

```bash
kubectl rollout history deployment/nginx-deploy
```

To rollback if needed:

```bash
kubectl rollout undo deployment/nginx-deploy
```

---

## 🔎 Example `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: gcr.io/google-samples/hello-app:1.0
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

Apply with `kubectl apply -f deployment.yaml`. The Service will get an EXTERNAL-IP (try `kubectl get svc web-svc`) which you can curl.

---

## 🔎 Essential Commands Cheat Sheet

```bash
# Cluster & Node Management (gcloud)
gcloud container clusters list
gcloud container clusters describe CLUSTER_NAME --zone ZONE
gcloud container clusters delete CLUSTER_NAME --zone ZONE

# Kubectl Basic Commands
kubectl get nodes,pods,deployments,services,hpa
kubectl describe node NODE_NAME
kubectl describe pod POD_NAME
kubectl logs POD_NAME

# Work with Deployments
kubectl create deployment NAME --image=IMAGE
kubectl apply -f file.yaml
kubectl scale deployment/NAME --replicas=N
kubectl autoscale deployment/NAME --min=1 --max=5 --cpu-percent=50    # HPA example【7†L991-L999】

# Rolling Updates
kubectl set image deployment/NAME CONTAINER=NEW_IMAGE
kubectl rollout status deployment/NAME
kubectl rollout undo deployment/NAME

# Service Exposure
kubectl expose deployment/NAME --type=LoadBalancer --port=80

# Node Pool Autoscaling (gcloud)
gcloud container node-pools create POOL_NAME --cluster CLUSTER_NAME \
    --zone ZONE --enable-autoscaling --min-nodes 1 --max-nodes 5【20†L864-L872】

# Node Auto-Upgrade (gcloud)
gcloud container node-pools update POOL_NAME --cluster CLUSTER_NAME \
    --enable-autoupgrade【26†L765-L773】
```

---

## 📊 GKE Features Demonstrated

- **Cluster Provisioning:** Standard (zonal) and Autopilot (regional) clusters.  
- **Node Pools:** Custom node pools (e.g., GPU pool) and autoscaling.  
- **Pod Lifecycle:** Deployments, ReplicaSets, Pods (imperative & declarative).  
- **Networking:** Services (`ClusterIP`, `LoadBalancer`), Ingress for L7 routing【22†L335-L343】.  
- **Autoscaling:** Horizontal Pod Autoscaler (pod scale)【7†L991-L999】 and Cluster Autoscaler (node scale)【20†L784-L793】.  
- **Updates:** Rolling updates/rollbacks of Deployments.  
- **Node Management:** Auto-upgrade and auto-repair enabled by default【26†L765-L773】【30†L1-L4】.  
- **CI/CD / Cloud Integration:** Using Container Registry/Artifact Registry for images, Cloud Build triggers (mention in CI context).  
- **Monitoring:** Cloud Monitoring (Stackdriver) for cluster and pod metrics.  

---

## 🎯 Skills Demonstrated

- **Kubernetes Fundamentals:** Pods, Deployments, Services, ConfigMaps/Secrets (mention), and declarative YAML.  
- **GKE-Specific Ops:** `gcloud` cluster/node management, understanding of Autopilot vs Standard.  
- **Scaling Strategies:** Manual scaling vs Autoscaling (HPA, Cluster Autoscaler).  
- **Networking:** Exposing services with Load Balancer and Ingress.  
- **Maintenance:** Rolling updates, node auto-upgrade/repair.  
- **Infrastructure as Code:** YAML manifests, `gcloud` scripting, possible Terraform mention.  
- **Troubleshooting:** Using `kubectl describe`, `kubectl logs`.  
- **CI/CD and Cloud Tools:** (Discussed as future work or notes: Cloud Build/Deploy, Helm, etc.)  

---

## 🗑️ Cleanup

When done, clean up the resources to avoid charges:

```bash
kubectl delete -f deployment.yaml
kubectl delete service/web-svc
gcloud container clusters delete my-cluster --zone us-central1-a
```

*(Replace names/zone as appropriate.)*

---

## 👨‍💻 Author

**Aqeel Saeed** – Cloud & DevOps Engineer

*Profile: [GitHub](https://github.com/aqeelsaeed138) | [LinkedIn](https://www.linkedin.com/in/yourprofile)*

