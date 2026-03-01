Assignment 2

## Phase 1: Environment Setup

### 1. Repository Configuration
**Assignment 1 Repo (App Code):**
```bash
git remote add origin [https://github.com/heracleo123/assignment1-app.git](https://github.com/heracleo123/assignment1-app.git) 
git add .
git commit -m "Deploy for Assignment2"
git branch -M main
git push -u origin main
Assignment 2 Repo (Manifests):

Bash
git remote add origin [https://github.com/heracleo123/k8s-assignment2-manifests.git](https://github.com/heracleo123/k8s-assignment2-manifests.git)
2. Connect to EC2 & Verify Tooling
Bash
ssh -i mykey.pem ec2-user@54.242.204.203
cd ~/k8s-manifests
ls -la mysql/
ls -la webapp/
Check Versions:

Bash
docker --version && kubectl version --client && kind version && git --version && aws --version
Phase 2: Cluster & Authentication
1. Initialize Cluster
Bash
kind create cluster --name assignment2-cluster
kubectl get nodes
kubectl cluster-info
2. Create Namespaces
Bash
kubectl apply -f namespace.yaml
3. AWS ECR Credentials
Configure CLI:

Bash
aws configure
# Use provided Access Key, Secret Key, and Session Token
Generate and Apply Secrets:

Bash
export ACCOUNT_ID=334566130486
export AWS_REGION=us-east-1
ECR_PASSWORD=$(aws ecr get-login-password --region $AWS_REGION)

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
Phase 3: Deployment Tasks
1. Standalone Pods (Task 2)
Bash
kubectl apply -f mysql/mysql-service.yaml
kubectl apply -f mysql/mysql-pod.yaml
kubectl apply -f webapp/webapp-pod.yaml

# Verify internal connectivity
kubectl get pod webapp-pod -n webapp-ns -o wide
docker exec -it assignment2-cluster-control-plane curl http://<POD_IP>:8080
2. ReplicaSets (Task 3)
Bash
kubectl apply -f mysql/mysql-replicaset.yaml
kubectl apply -f webapp/webapp-replicaset.yaml
kubectl get pods --show-labels -n webapp-ns
3. Deployments (Task 4)
Bash
kubectl apply -f mysql/mysql-deployment.yaml
kubectl apply -f webapp/webapp-deployment.yaml

# Cleanup manual resources to allow Deployment to manage pods
kubectl delete rs webapp-rs -n webapp-ns
kubectl delete pod webapp-pod -n webapp-ns
kubectl get pods -n webapp-ns -w
Phase 4: Networking & Exposure (Task 5)
1. NodePort Service
Bash
kubectl apply -f webapp/webapp-service.yaml
kubectl get svc -n webapp-ns
2. External Access
Bash
# Bridge cluster to EC2 public IP
kubectl port-forward --address 0.0.0.0 svc/webapp-service 30000:8080 -n webapp-ns
# Access via browser: http://<EC2_PUBLIC_IP>:30000
Phase 5: Rolling Updates (Task 6)
1. Image Sideloading
Bash
# Tag existing image as v1 and load into kind
docker tag [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:latest](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:latest) [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1)
kind load docker-image [334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1](https://334566130486.dkr.ecr.us-east-1.amazonaws.com/webapp:v1) --name assignment2-cluster
2. Execute Update
Bash
# Update webapp-deployment.yaml image tag to :v1, then apply
kubectl apply -f webapp/webapp-deployment.yaml
kubectl rollout status deployment/webapp-deployment -n webapp-ns
3. Verify Version
Bash
kubectl describe pod <NEW_POD_NAME> -n webapp-ns | grep Image
Cleanup
Bash
kind delete cluster --name assignment2-cluster
