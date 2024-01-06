create a t2.micro server, install helm, aws cli, run aws configure, install eksctl , install kubectl

documentation for installing:
https://phoenixnap.com/kb/install-helm
https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eksctl.html
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html 
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

Clone the repo:
run following commands:


1. eksctl create cluster --name two-tier-app --region ap-south-1 --node-type t2.medium --nodes-min 1 --nodes-max 2
eksctl get cluster(aws configure (access key nd secret key firse if u cnat get clusters))

2. aws eks update-kubeconfig --name two-tier-app
3.  kubectl create namespace two-tier-flask-mysql
  
4. Configure OIDC (so that worker nodes can communicate with aws services) follow: https://github.com/iam-veeramalla/aws-devops-zero-to-hero/blob/main/day-22/configure-oidc-connector.md
   a) export cluster_name=two-tier-app
   b) oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
   c) eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
  
5. Configure alb-ingress-controller (as we are using ingress resource to route the traffic to pods) follow: https://github.com/iam-veeramalla/aws-devops-zero-to-hero/blob/main/day-22/alb-controller-add-on.md
   
   a) curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
   b) aws iam create-policy     --policy-name AWSLoadBalancerControllerIAMPolicy     --policy-document file://iam_policy.json
   c) eksctl create iamserviceaccount   --cluster=<your-cluster-name>   --namespace=kube-system   --name=aws-load-balancer-controller   --role-name AmazonEKSLoadBalancerControllerRole   --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy   --approve
   d) helm repo add eks https://aws.github.io/eks-charts
   e) helm repo update eks
   
g)          helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
	  --set clusterName=two-tier-app \
	  --set serviceAccount.create=false \
	  --set serviceAccount.name=aws-load-balancer-controller \
	  --set region=<region> \
	  --set vpcId=<vpc-id>
  
  kubectl get deployment -n kube-system aws-load-balancer-controller
 
 
6.  kubectl apply -f flask-ingress.yaml -n two-tier-flask-mysql
7.  kubectl get ingress -n two-tier-flask-mysql (check if u can see the address of LB if yes then alb is configured check in aws console (load balancerss))
 
8.Configure EBS driver(ebs driver will provision a dynamic volume (check volumes on aws console). it willbe configure using a default stroageClass that is gp2 as we didnt define any storageclass . and whatever we have  mention in pvc.yml it will claim that volume ) 
(follow: https://github.com/iam-veeramalla/three-tier-architecture-demo/blob/master/EKS/05-ebs-csi-driver.md)

 eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster <YOUR-CLUSTER-NAME> \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
	
 eksctl create addon --name aws-ebs-csi-driver --cluster two-tier-app --service-account-role-arn arn:aws:iam::<account:id>:role/AmazonEKS_EBS_CSI_DriverRole --force
 kubectl get storageClass
 kubectl get storageClass -n two-tier-flask-mysql
 
 (go to aws console and check Volumes)
  
9. aws eks describe-cluster --name two-tier-app --region ap-south-1
  
10. Apply all manifests (leave mysql-deployment.yaml because we are deploying mysql-statefulset.yml)
  

   kubectl apply -f flask-deployment.yml -f flask-ingress.yaml -f flask-service.yml -f mysql-configMaps.yml -f mysql-db-query-configMaps.yml -f mysql-pvc.yaml -f mysql-secrets.yml -f mysql-service.yaml -f mysql-statefulset.yaml -n two-tier-flask-mysql
   kubectl get all -n two-tier-flask-mysql
   sudo nano flask-deployment.yml (edit the clusterIP of mysql service and apply again)
   kubectl apply -f flask-deployment.yml -f flask-ingress.yaml -f flask-service.yml -f mysql-configMaps.yml -f mysql-db-query-configMaps.yml -f mysql-pvc.yaml -f mysql-secrets.yml -f mysql-service.yaml -f mysql-statefulset.yaml -n two-tier-flask-mysql
   kubectl get all -n two-tier-flask-mysql
  
check the load balancer ip(app is ruuning)
kubectl get ingress -n two-tier-flask-mysql

kubectl exec -it <mysql-pod-name> -n <namespace> -- /bin/bash
 ----------------------------------------------------------------------------------------------------------------------- 
to add observability of your cluster and send logs to cloud watch :
  
  1. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-prerequisites.html
  attach this policy to your worker-node
  
  2. install the Amazon CloudWatch Observability EKS add-on(follow: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-EKS-addon.html)
  aws iam attach-role-policy \
--role-name <my-worker-node-role> \
--policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
--policy-arn arn:aws:iam::aws:policy/AWSXrayWriteOnlyAccess


 to get role-name of your worker-node run
  eksctl get nodegroup --cluster two-tier-app
  -->ASG is nodegroup-name
  aws eks describe-nodegroup --cluster-name two-tier-app --nodegroup-name ng-5dca75a6
  aws eks describe-nodegroup --cluster-name two-tier-app --nodegroup-name ng-5dca75a6 | grep -A 1 "nodeRole"
  
  3. aws eks create-addon --cluster-name my-cluster-name --addon-name amazon-cloudwatch-observability
  
  4. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-metrics.html
  5. https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs-FluentBit.html
  6. kubectl get pods -n amazon-cloudwatch

