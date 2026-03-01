# Assignment 2: Kubernetes Deployment & Networking Report
**Project:** ElectroTech E-Commerce Platform
**Program:** Cloud Architecture and Administration, Seneca Polytechnic

## Project Overview
This repository contains the Kubernetes manifest files and deployment documentation for the **ElectroTech** e-commerce platform. The project demonstrates the successful deployment of a highly available, multi-tier architecture using a local `kind` Kubernetes cluster, featuring a Python frontend web application and a MySQL backend database.

---

## Technical Questions & Architecture Analysis

### 1. K8s API Server IP (Task 1.a)
**Q: What is the IP of the K8s API server in your cluster?**
**A:** The Kubernetes API server (control plane) for my `kind` cluster is running at `127.0.0.1`. Specifically, it was bridged to my EC2 host at `https://127.0.0.1:42497`.

### 2. Pod Networking (Task 2.b)
**Q: Can both applications listen on the same port inside the container? Explain your answer.**
**A:** Yes, they technically can. In Kubernetes, every Pod is assigned its own unique IP address and isolated network namespace. Because the MySQL database and the ElectroTech web application are deployed in completely separate Pods with different IP addresses, they could both safely listen on the same port without any network collisions.

### 3. ReplicaSet Governance (Task 3)
**Q: Is the pod created in step 2 governed by the ReplicaSet you created? Explain.**
**A:** Yes, the standalone pod is governed by the new ReplicaSet. Kubernetes ReplicaSets use "Label Selectors" to manage state. Because the standalone pod was given the label `app: employees`, the newly created ReplicaSet identified it, recognized the matching label, and "adopted" it into its desired count of 3 replicas. 

### 4. Deployments vs. ReplicaSets (Task 4.b)
**Q: Is the replicaset created in step 3 part of this deployment? Explain.**
**A:** No, the manual ReplicaSet is completely ignored by the Deployment. When a Deployment is created, it automatically generates and manages its *own* unique ReplicaSet (identified by an alphanumeric hash appended to its name, such as `webapp-deployment-7464c77dd4`). It does not assume ownership of manually created ReplicaSets, even if they share the same label selectors.

### 5. Service Types (Task 7 / 6.a)
**Q: Explain the reason we are using different service types for the web and MySQL applications.**
**A:** Different service types are used to enforce security and control traffic flow:
* **Web Application (NodePort):** This service type is used to expose the frontend application to external traffic, allowing users to access the ElectroTech website via a web browser.
* **MySQL Application (ClusterIP):** This service type restricts access entirely to within the Kubernetes cluster. It ensures the database remains secure and is never exposed to the public internet, accessible only by the internal web application.

---

## Real-World Troubleshooting & Resolutions

Throughout the deployment process for the ElectroTech platform, I encountered and resolved several multi-layered infrastructure, configuration, and orchestration challenges:

### 1. Environment & Workflow Challenges
* **Disk Space Exhaustion:** * *Issue:* The AWS Cloud9 environment ran out of disk space, which caused Docker image pull and build failures.
  * *Resolution:* Executed a system cleanup by pruning unused Docker images and removing temporary files, successfully freeing up 2.18GB of storage to allow the builds to continue.
* **Git vs. SCP File Transfer Errors:** * *Issue:* Initially, trying to use `scp` to copy manifest files from the Cloud9 IDE to the EC2 instance resulted in "Permission denied" errors because it conflicted with Git's hidden read-only `.git` tracking files.
  * *Resolution:* Abandoned `scp` in favor of a standard Git workflow. Pushed the manifests from Cloud9 to GitHub, and used `git pull` on the EC2 instance to cleanly sync the files and maintain the required commit history.

### 2. Container Registry & Authentication Issues
* **ECR Authentication (401 Unauthorized):**
  * *Issue:* Pods failed to pull images from Amazon ECR, resulting in 401 Unauthorized errors.
  * *Resolution:* Generated AWS ECR login credentials and created Kubernetes secrets of type `docker-registry`. Appended `imagePullSecrets` to all pod, ReplicaSet, and Deployment manifests.
* **AWS ECR Token Expiration (`ImagePullBackOff`):**
  * *Issue:* Later in the deployment, pods failed to initialize again because the AWS ECR authentication token inside the `regcred` secret had exceeded its 12-hour lifespan and expired.
  * *Resolution:* Deleted the expired secrets in both namespaces, generated a fresh token via the AWS CLI, and re-created the `docker-registry` secrets. 
* **Kind Cluster IAM Limitations:**
  * *Issue:* The local `kind` cluster did not inherently possess the IAM permissions required to pull images directly from AWS ECR, causing `ErrImagePull`.
  * *Resolution:* Used the `kind load docker-image` command to manually sideload the images from the EC2 host directly into the Kind cluster's control-plane node, and updated the manifests to use `imagePullPolicy: IfNotPresent` to bypass external network dependencies.

### 3. Application & Database Configuration
* **Database Connection Typo (`CrashLoopBackOff`):** * *Issue:* Web application pods crashed on startup. Logs (`kubectl logs`) revealed a Python `ConnectionRefusedError`, indicating the app was routing to `localhost` instead of the database service.
  * *Resolution:* Investigated the `app.py` source code and identified a mismatch in expected environment variable names. Updated the manifest to pass `DBHOST` instead of `DB_HOST`.
* **Missing Database Port (`CrashLoopBackOff`):**
  * *Issue:* The frontend pods crashed again with a Python `TypeError`. Logs indicated this was caused by a missing `DBPORT` environment variable, returning a `NoneType` during the integer conversion.
  * *Resolution:* Updated the pod and deployment manifest files to ensure all required database connection variables (`DBHOST`, `DBUSER`, `DBPWD`, `DBPORT`, `DATABASE`) were explicitly injected into the container environment.

### 4. Kubernetes Orchestration
* **Deployment Scaling Conflicts:**
  * *Issue:* A newly applied Deployment failed to spin up its own pods because a manual ReplicaSet from a previous step was already fulfilling the required label selector quota (`app: employees`).
  * *Resolution:* Deleted the manual resources (`webapp-rs` and `webapp-pod`) to clear the label conflict and force the Kubernetes Deployment controller to spin up its own uniquely hashed ReplicaSet and pods.
