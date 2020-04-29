---
title: EKS Fargate Demo
author: Nick Brandaleone
patat:
  images:
    backend: iterm2
...

# EKS Fargate

## Agenda
 
- Create an EKS cluster
- Setup IAM Roles for Service Accounts (IRSA)
- Create a RBAC role, binding and Service Account
    
    > * Deploy the ALB ingress controller
    > * Create a Kubernetes deployment, service and ingress
    > * Verify that everything is working on Fargate managed containers

## Overall Architecture

![](/Users/nbrand/src/talks/patat/images/ALB-Ingress-Controller-Fargate-architecture.png)


## Create the EKS cluster with Fargate
*Aleady completed, since it takes about 15 minutes*

    export AWS_REGION=us-east-2
    export CLUSTER_NAME=eks-fargate-alb-demo

    eksctl create cluster --name $CLUSTER_NAME --region $AWS_REGION --fargate

## Verify we have an EKS cluster and no instances running

    aws eks --region $AWS_REGION list-clusters

    aws ec2 --region $AWS_REGION describe-instances \
    --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value[], 
    InstanceId, InstanceType, State.Name]' --output table

# Examine the Fargate profile

## AWS CLI Commands
[comment]:    PROFILE=$(aws eks --region $AWS_REGION list-fargate-profiles
[comment]:    --cluster-name $CLUSTER_NAME | jq -r .fargateProfileNames[])

    aws eks describe-fargate-profile --region $AWS_REGION \
    --cluster $CLUSTER_NAME --fargate-profile-name $PROFILE

# Create OIDC provider for IRSA (IAM Roles for Service Accounts)

## AWS CLI Commands
*Aleady completed*

    eksctl -r $AWS_REGION utils associate-iam-oidc-provider \
    --cluster $CLUSTER_NAME --approve

## Create IAM Policy which limits what pod (ALB controller) can do
*See blog post for actual policy recommendation*

Actions allowed:

- ACM certificates
- EC2
- Elastic LoadBalancing
- WAF

## Create role, and bind it to a service account
*Already completed*

<!--    STACK_NAME=eksctl-$CLUSTER_NAME-cluster
    
    export VPC_ID=$(aws --region $AWS_REGION cloudformation \
    describe-stacks --stack-name "$STACK_NAME" | \
    jq -r '[.Stacks[0].Outputs[] | {key: .OutputKey, value: .OutputValue}] \
    | from_entries' | jq -r '.VPC')

   AWS_ACCOUNT_ID=$(aws sts get-caller-identity | jq -r '.Account')
-->
RBAC role limits what a pod (ALB controller) can do from a Kubernetes perspective

    kubectl apply -f rbac-role.yaml

## Create the service account, and link it to IAM permissions

*Already completed*

    eksctl create iamserviceaccount \
    --region $AWS_REGION \
    --name alb-ingress-controller \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/ALBIngressControllerIAMPolicy \
    --approve

## Deploy the ingress controller
[comment]: envsubst < alb-ingress-controller.yaml > alb-controller.yaml
The controller configures an ALB in response to K8 ingress objects

    kubectl apply -f alb-controller.yaml

# Deploy a sample application (nginx)

## Create Kubernetes deployment, service, and ingress
*3 nginx pods*
*service is of type NodePort*
*service has annotation:* **alb.ingress.kubernetes.io/target-type: ip**

    kubectl apply -f nginx-deployment.yaml
    kubectl apply -f nginx-service.yaml
    kubectl apply -f nginx-ingress.yaml

# Examine the ingress spec

## Kubernetes ingress object

    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: "nginx-ingress"
      namespace: "default"
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
      labels:
        app: nginx-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /*
            backend:
              serviceName: "nginx-service"
              servicePort: 80

## Check on the health of your pods and LoadBalancer
<!--
LOADBALANCER_PREFIX=$(kubectl get ingress nginx-ingress -o json | \
jq -r '.status.loadBalancer.ingress[0].hostname' | cut -d- -f1)
TARGETGROUP_ARN=$(aws --region $AWS_REGION elbv2 \
describe-target-groups | jq -r '.TargetGroups[].TargetGroupArn' \
| grep $LOADBALANCER_PREFIX)
aws --region $AWS_REGION elbv2 describe-target-health \
--target-group-arn $TARGETGROUP_ARN | \
jq -r '.TargetHealthDescriptions[].TargetHealth.State'
-->

    kubectl get ingress nginx-ingress
    kubectl get pods -o wide
    kubectl get nodes

# Verify the application works from the Internet

## Clean up

    kubectl delete ingress/nginx-ingress
    kubectl delete deployment/nginx-deployment
    kubectl delete service/nginx-service
    kubectl delete deploy/alb-ingress-controller -n kube-system

    eksctl delete cluster $CLUSTER_NAME --region $AWS_REGION

## Summary

- EKS with Fargate *eliminates* heavy lifting of managing servers/instances
- You can run EKS clusters with both EC2 and Fargate worker nodes
- It is important to use resource request/limit (make them the same for Fargate)
- There are some downsides to Fargate
	> * Less control over instance choice
	> * Only works with ALB LoadBalancers
	> * Slower pod start-up time (although faster than EC2 launch)
  > * No Daemon sets
  > * Limited volume space
  > * No ability to mount shared file systems
	> * Different pricing model

## Appendix

- Blog: https://aws.amazon.com/blogs/containers/using-alb-ingress-controller-with-amazon-eks-on-fargate/
- AWS ALB controller docs: https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html
- ALB controller: https://github.com/kubernetes-sigs/aws-alb-ingress-controller
- EKS Fargate: https://docs.aws.amazon.com/eks/latest/userguide/fargate.html
- IAM Roles for Service Accounts: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html

# The presentation is over
