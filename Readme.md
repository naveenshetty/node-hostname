# Node Hostname Application

This repository contains the Node.js application for handling hostname requests. The application has been Dockerized, pushed to Docker Hub, and deployed to an EKS cluster using Helm and ArgoCD for continuous deployment.

## Table of Contents

- [Technologies Used](#technologies-used)

## Technologies Used

- **Node.js**: JavaScript runtime for building the application.
- **Docker**: For containerizing the Node.js application.
- **Helm**: For managing Kubernetes charts.
- **ArgoCD**: For Continuous Deployment (CD) to AWS EKS.
- **AWS EKS**: For managing Kubernetes clusters in AWS, ASG in place for high availablity.

- **Some picture:** 
- <img src="image/img.png" alt="UI" width="400">
- <img src="image/img_1.png" alt="UI-1" width="400">
- <img src="image/img_2.png" alt="argocd" width="400">


- **Steps to follow creation of cluster from eksctl:** 
`eksctl create cluster --name=node-hostname \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup`

- **Associate cluster to iam oidc provider, this will help by cluster to interact with other AWS resources 
` eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster node-hostname \
    --approve`

- **Create Nodegroup:**
`eksctl create nodegroup --cluster=node-hostname \
                        --region=us-east-1 \
                        --name=node-hostname-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking`

# Update ./kube/config file
aws eks update-kubeconfig --name node-hostname

**Require when observability come into picture like installing elasticserch on cluster and attaching EBS volume into it so for this reason we need this.  
**Create IAM Role for Service Account:**
`eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster node-hostname \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve`
* This command creates an IAM role for the EBS CSI controller.
* IAM role allows EBS CSI controller to interact with AWS resources, specifically for managing EBS volumes in the Kubernetes cluster.
* We will attach the Role with service account

**Retrieve IAM Role ARN**
`ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)`
* Command retrieves the ARN of the IAM role created for the EBS CSI controller service account.

**Deploy EBS CSI Driver**
`eksctl create addon --cluster node-hostname --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force`
* Above command deploys the AWS EBS CSI driver as an addon to Kubernetes cluster.
* It uses the previously created IAM service account role to allow the driver to manage EBS volumes securely.

** Installing elasticserch**

`kubectl create namespace obser
`
**Install Elasticsearch on K8s**
`helm install elasticsearch
--set replicas=1
--set volumeClaimTemplate.storageClassName=gp2
--set persistence.labels.enabled=true elastic/elasticsearch -n obser`

**Export Elasticsearch CA Certificate**
* This command retrieves the CA certificate from the Elasticsearch master certificate secret and decodes it, saving it to a ca-cert.pem file.
`kubectl get secret elasticsearch-master-certs -n obser -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem
`

**Create Secret for Elasticsearch TLS**
* Creates a Kubernetes Secret in the tracing namespace, containing the CA certificate for Elasticsearch TLS communication.
`kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n obser
`
**Retrieve Elasticsearch Username & Password**
# for username
kubectl get secrets --namespace=obser elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=obser elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d

* Retrieves the password for the Elasticsearch cluster's master credentials from the Kubernetes secret.

**Install Opentelemetry-collector**
* helm install otel-collector open-telemetry/opentelemetry-collector -n obser --values otel-collector-values.yaml

**Install prometheus**
`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install  prometheus prometheus-community/prometheus -n obser --values prometheus-values.yaml`

**Deploy the application either from ArgoCD using helm chart or manually using below**
`kubectl apply -k k8s/
`
**WE can use some random script to create load, here I did not use**
**Access the UI of Prometheus**
`kubectl port-forward svc/prometheus-server 9090:80 -n obser
`