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



