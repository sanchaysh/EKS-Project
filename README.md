                                        EKS Project with ALB Ingress
I am excited to share, I did of late to deploy a Game application on EKS cluster and used ALB Controller as ingress to expose the applications to the outside world.

Here are the AWS Services & Tools I used to achieve this.

AWS Services

1.EKS - Managed Kubernetes Service. 
2.Fargate - For running Pods. 
3.ECR - To hold Container images. 
4.Core DNS- Service registry & Discovery. 
5.Kube-Proxy - To implement IPVtables 
6.AWS VPC CNI -To implement Pod networking. 
7.AWS IAM- OIDC Identity Provider.

Tools.

1.AWS CLI - To configure access to AWS. 
2.eksctl - For creating, managing & Deleting EKS Cluster. 
3.Kubectl- Used to interact with the Cluster. 
4.Helm - To install ALB ingress Controller.

High Level Steps.

1.EKS Cluster Creation with Fargate.

eksctl create cluster --name demo-cluster --region ap-south-1 --fargate

You can also see in CloudFormation if the Cluster is created.

Update the Kubeconfig file to interact with EKS Cluster.

aws eks update-kubeconfig --name demo-cluster --region ap-south-1

Creation of Fargate Profile.
By default it creates a Fargate profile (fp-default) which allows you to run pods only on default and kube-system namespace). If you need to run your application in a different namespace so you need to create a new Fargate profile.

eksctl create fargateprofile
--cluster demo-cluster
--region ap-south-1
--name alb-sample-app
--namespace game-2048

You can see the Fargate Profile created in EKS Console under Compute Tab.

3.Manifest Files for Deployment of the application along with Ingress.

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

4.Configure IAM as OIDC Provider to authenticate to EKS cluster.

eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve

5.Creation of IAM Policy for ALB Controller.

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

Create IAM Policy

aws iam create-policy
--policy-name AWSLoadBalancerControllerIAMPolicy
--policy-document file://iam_policy.json

6.Create the Service account & IAM role and attach the Policy to allow ALB controller Pods to talk to AWS Application Load Balancer.

eksctl create iamserviceaccount
--cluster=
--namespace=kube-system
--name=aws-load-balancer-controller
--role-name AmazonEKSLoadBalancerControllerRole
--attach-policy-arn=arn:aws:iam:::policy/AWSLoadBalancerControllerIAMPolicy
--approve

You can also see the Service account created in Cloudformation.

Note : Replace Cluster Name & Account ID

Deploy ALB Controller on EKS Cluster.
helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
-n kube-system
--set clusterName=demo-cluster
--set serviceAccount.create=false
--set serviceAccount.name=aws-load-balancer-controller
--set region=ap-south-1
--set vpcId= <yourvpcId>

Note: Replace Cluster name, Region and VPC ID.

Verify if Deployment is running

kubectl get deployment -n kube-system aws-load-balancer-controller

Locate Load balancer DNS name to access test the Application.
Kubect get ing -n game-2048

Copy the Load balancer DNS Name and paste it in the browser.

http://k8s-game2048-ingress2-bcac0b5b37-9014610.ap-south-1.elb.amazonaws.com/

You may need to wait for 5-10 minutes for load balancer to fully provision.

http://k8s-game2048-ingress2-bcac0b5b37-9014610.ap-south-1.elb.amazonaws.com/

You can access your Game Application now.
