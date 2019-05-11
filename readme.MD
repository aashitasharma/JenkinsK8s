## Description
the purpose of this repo is to guide you to a very simple cluster creation process of Jenkins using kubernetes and Nginx on top of AWS. By the end of this guide you should have a working Jenkins cluster that can scale up or down based on your needs. Please note I intentionally left our DNS set up as I thought that would be a bit difference depending on how or who you registered your domains with.

## Create cluster

Assumptions

IAM role with the following permissions
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
AmazonVPCFullAccess


## Optional step to create a user name kops to handle our cluster creating process

a.- run the following commands
	aws iam create-group --group-name kops
	aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess --group-name kops
	aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess --group-name kops
	aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --group-name kops
	aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/IAMFullAccess --group-name kops
	aws iam attach-group-policy --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess --group-name kops
	aws iam create-user --user-name kops
	aws iam add-user-to-group --user-name kops --group-name kops
	aws iam create-access-key --user-name kops


## Create Cluster

1.- run the following command to create S3 space

    bucket_name=<name-of-bucket>
	aws s3api create-bucket \
	--bucket ${bucket_name} \
	--region us-east-1

2.- run this to enable state to s3

	aws s3api put-bucket-versioning --bucket ${bucket_name} --versioning-configuration Status=Enabled

3.- run this to save values we are going to use later

	export KOPS_CLUSTER_NAME=<name of my cluster>.local
	export KOPS_STATE_STORE=s3://${bucket_name}

4 .-  run to create cluster config. Please adjust master-size, node-count, node-size, zones as needed. The more resource you put here the  more you will pay

	kops create cluster \
	--master-size=t2.micro \--node-count=1 \
	--node-size=t2.micro \
	--zones=us-east-1a \
	--name=${KOPS_CLUSTER_NAME} 

5.-  run the following to create cluster

	kops update cluster --name ${KOPS_CLUSTER_NAME} --yes

6.- this may take from 5~8 minutes  you can run this command to check status of cluster

	kops validate cluster

## Optional Steps Setup Dashboard UI 

  a.- run the following to enable dashboard ui

   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard.yaml

   b.- navigate to root folder and run the following command run the following to create user

     Kubectl apply -f dashboard-adminuser.yaml && Kubectl apply -f role-binding-user.yaml 

   c.-  run the following to grab admin user token to login. Copy this token as this will be your acces key

    kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

  d.- run the follwing to proxy into the cluster using dashboard UI

      kubectl proxy

  e.- Navigate to following url and login     http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/ 
 

## Add Helm charts to rapid development

   a.- run this if you don’t have helm already installed

	brew install kubernetes-helm

   b.- run the following command to create service account

        Kubectl apply -f  https://gist.githubusercontent.com/chukaofili/0389b3e9cb2b391faa5cf68660a481d3/raw/44643d2d5e735d34a7731f6cc8cab04d5f249b8e/helm-rbac-config.yaml

   c.- run this to make sure service account named tiller has been created in your cluster. Which basically mean get me all the service account in the cube-system namespace

      kubectl get sa -n kube-system

   d.- initialize helm by running the following command. You should get "happy helming" message if successful.
   
      helm init —service-account tiller

## Add Nginx ingress controller using helm charts
    
    a.-  Navigate to the NGINX  directory and run the following command to create nginx-ingress namespace to separate from the ones from our pods

          kubectl create -f   ingress-nginx.json
         
    b.-  Navigate to the NGINX directory and search for ##replace with your specification##. Once this is sanitized with your specs you can run the following

     	helm install stable/nginx-ingress --name <name-of-your-controller> --namespace ingress-nginx -f values.yaml 

## Create Pods
    a.- Sanitize Jenkins-deployment.yaml. Search for ##replace with your specs##

    b.- Navigate to Jenkins directory and run the following command to create our Jenkins instance
        kubectl apply -f Jenkins-deployment.yaml
   
    c.- this will take about 5 minutes or so but you can always check on status of deployment by either logging into the dashboard UI or running the following command
       Kubectl get pods                   ## this will give you pod creating status in the default namespace        kubectl get service              ## this will give you all the services created in the default namespace . If successful it will give you the external address you can reach your app         kubectl get ingress              ## will give you all ingress resources created in the default namespace 
