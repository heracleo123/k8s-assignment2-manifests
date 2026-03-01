Kubernetes Deployment & Infrastructure Report
### Assignment 2 - Cloud Architecture and Administration | Seneca Polytechnic

---

## 🚀 Phase 1: Environment Setup

### 1. Repository Configuration
**Application Source (Assignment 1):**
```bash
git remote add origin [https://github.com/heracleo123/assignment1-app.git](https://github.com/heracleo123/assignment1-app.git) 
git add .
git commit -m "Deploy for Assignment2"
git branch -M main
git push -u origin main
K8s Manifests (Assignment 2):Bashgit remote add origin [https://github.com/heracleo123/k8s-assignment2-manifests.git](https://github.com/heracleo123/k8s-assignment2-manifests.git)
2. Infrastructure ValidationBashssh -i mykey.pem ec2-user@54.242.204.203
cd ~/k8s-manifests

# Verify toolchain versions
docker --version && kubectl version --client && kind version && git --version && aws --version
🛠 Phase 2: Cluster & Authentication1. Initialize Kind ClusterBashkind create cluster --name assignment2-cluster
kubectl apply -f namespace.yaml
2. AWS ECR AuthenticationBashaws configure
# Enter Access Key, Secret Key, and Session Token as prompted

export ACCOUNT_ID=334566130486
export AWS_REGION=us-east-1
ECR_PASSWORD=$(aws ecr get-login-password --region $AWS_REGION)

# Apply registry secrets to both namespaces
kubectl create secret docker-registry regcred \
  --docker-server=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  --namespace=webapp-ns

kubectl create secret docker-registry regcred \
  --docker-server=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com \
  --docker-username=AWS \
  --docker-password="$ECR_PASSWORD" \
  --namespace=mysql-ns
📦 Phase 3: Deployment Tasks1. Standalone Pods (Task 2)Bashkubectl apply -f mysql/mysql-service.yaml
kubectl apply -f mysql/mysql-pod.yaml
kubectl apply -f webapp/webapp-pod.yaml

# Verify internal connectivity (Must run from inside Kind node)
kubectl get pod webapp-pod -n webapp-ns -o wide
docker exec -it assignment2-cluster-control-plane curl http://<POD_IP>:8080
2. ReplicaSet Adoption (Task 3)Bashkubectl apply -f mysql/mysql-replicaset.yaml
kubectl apply -f webapp/webapp-replicaset.yaml
kubectl get pods --show-labels -n webapp-ns
3. Managed Deployments (Task 4)Bashkubectl apply -f mysql/mysql-deployment.yaml
kubectl apply -f webapp/webapp-deployment.yaml

# Clear manual resources to resolve label selector conflicts
kubectl delete rs webapp-rs -n webapp-ns
kubectl delete pod webapp-pod -n webapp-ns
🌐 Phase 4: Networking & Exposure (Task 5)1. External Access (NodePort)Bashkubectl apply -f webapp/webapp-service.yaml

# Bridge cluster to EC2 public IP for browser access
kubectl port-forward --address 0.0.0.0 svc/webapp-service 30000:8080 -n webapp-ns
# Access via: http://<EC2_PUBLIC_IP>:30000
🔄 Phase 5: Rolling Updates (Task 6)1. Sideloading the v1 ImageBashdocker tag [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:latest](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:latest) [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1)
kind load docker-image [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1) --name assignment2-cluster
2. Execute RolloutBash# Update webapp-deployment.yaml image tag to :v1
kubectl apply -f webapp/webapp-deployment.yaml
kubectl rollout status deployment/webapp-deployment -n webapp-ns
📝 Troubleshooting & Post-MortemIssueResolutionDisk Space ExhaustionCleaned up Docker system images, freeing 2.18GB of space.SCP Permission DeniedAbandoned SCP for a proper Git push/pull workflow to maintain commit history.ECR Token ExpirationRegenerated AWS tokens and recreated regcred secrets after 12-hour expiration.Python TypeErrorFixed missing DBPORT environment variable in the deployment manifest.Deployment IdleResolved label selector conflict by deleting manual ReplicaSets that were "stealing" pods.🧹 CleanupBashkind delete cluster --name assignment2-cluster
