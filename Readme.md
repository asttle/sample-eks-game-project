Pre-requistites 
Install eksctl - command line tool to install eks cluster using cloudformation templates
 (setting up vpc,private and public subnets entirely taken care)

Install kubectl - command line tool to interact with kubernetes cluster

Install aws cli - command line tool to interact with aws services

Configure aws cli with access key and secret key

eksctl create cluster --name demo-cluster --region eu-west-2 --fargate --profile=asttle-personal

openid connect - identity provider integration 
You can use IAM or use this url, you can also integrate with other identity providers like okta, ldap for users,serviceaccounts
management to access eks cluster

Setup context for your cluster to access it via kubectl so you can switch between clusters and add the kubeconfig to your local to retreive data about cluster from local

aws eks update-kubeconfig --name demo-cluster --region eu-west-2 --profile=asttle-personal

Create a separate namespace for your app with fargateprofile
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region eu-west-2 \
    --name alb-sample-app \
    --namespace game-2048 \
    --profile=asttle-personal

Create the game from yaml url
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

eloborated as individual files

ingress resource 
kubectl get ingress -n game-2048
you can see address is empty which means ingress is not listening to any controller, once we have ingress controller in place, we will have address bound to it 

Next we can create ingress controller -> which will be bound to a ingress resource which will create the alb for us
This is not only creation of load balancer but target group specification, port, 

SO next step is to create ALB controller, but before that we need to allow kubernetes pod to create aws resource(ALB) so we need to associate oidc identity provider to allow pod to crate a ALB resource
This is like providing the trust for kubernetes service accounts to utlise IAM roles and policies for auth an authorization purposes. 

eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve --profile=asttle-personal

Use the iam policy created which can be retrieved from ofical alb controller documentation and create iam policy

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json \
    --profile=asttle-personal

next create an iam role and attach policy to it 
Instead of creating role separately we can create serviceaccount directly and map role and policy attached to it will create serviceaccount resource in given namepsace
eksctl create iamserviceaccount \
  --cluster=demo-cluster\
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::306903349732:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --profile=asttle-personal

  Next install the controller
  helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=eu-west-2\
  --set vpcId=vpc-0de0a3d2264bf8eb2

  kubectl get deployment -n kube-system aws-load-balancer-controller

  This will create controller in 2 availability zones 

  kubectl get ingress -n game-2048
  Now you can see address is filled in alb url which states alb controller has listeneed to this ingress and created the alb on this address

  Go to browser and check the url and there we go! Game on!


Once everything is done delete the cluster created

eksctl delete cluster --name demo-cluster --region eu-west-2 --profile=asttle-personal





