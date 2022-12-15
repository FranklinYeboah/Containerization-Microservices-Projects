=======Deploy A Stateless Sample Application on an EKS Cluster========

1) SETUP YOUR ENVIRONMENT  
--> Launch an Ubuntu 18.04 or a Cloud9 Instance 
--> Create an EC2 Service Role with "Administrator access and attach to the EC2 Instance"
--> Create a user with programatic access and assign Administratir Policy
--> Download the `sample-stateless-app` folder or `clone the repository` 
    - https://github.com/awanmbandi/stateless-web-app.git
--> Install git (On the VM, If you launch your Ubuntu 18 from AWS Marketplace)
--> Upload folder on your Cloud9 Environment or better still clone repository

A) INSTALL PIP AND PIP3 python libaries
--> sudo apt update -y
--> sudo apt install python3-pip -y
--> pip3 --version
--> sudo apt install python-pip -y
--> pip --version
 

B) Install the Bellow Tools  
Install AWSCLI 

--> pip3 install --user awscli

--> mkdir ~/.aws 

--> touch ~/.aws/credentials 
    > Provide Access and Secrete Access Keys       (Please Make sure you run this command and configure your cli) 

--> vi ~/.aws/credentials 

[User_Profile]
aws_access_key_id=.....
aws_secret_access_key=.....
region=us-east-1
output=json

================================================================================================================

Install eksctl 

--> curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

--> sudo mv /tmp/eksctl /usr/local/bin

--> eksctl version 


Install kubectl binary 

--> sudo apt-get update 

--> sudo apt-get install -y apt-transport-https

--> curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

--> sudo touch /etc/apt/sources.list.d/kubernetes.list 

--> echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list 

--> sudo apt-get update 

--> sudo apt-get install -y kubectl

--> kubectl version --short --client 

--> kubectl


2) SET UP EKS CLUSTER 
# use eksctl to create EKS cluster 

--> eksctl create cluster --help 

## Cluster creation 
create cluster by using prebuild template designed and developed by aws wich comes with the default configurations 

NOTE NOTE NOTE: USE: https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html

- eksctl create cluster --name CLUSTER_NAME --region REGION

NOTE: 
  * Check CloudFormation for the Newly created Stack with the resources
  * Also open the EKS Console to confirm the Cluster is getting created

## post-install check 
eksctl also creates the `config file` for `_kubectl_.` This means we can immediately fire up a check like: 
#Check Cloudformation for PROGRESS 

THE `/home/ubuntu/.kube/config` HAS BEEN CREATED SO WE CAN NOW `START RUNNING kubtcl commands`

--> kubectl get nodes 

3) AWS EKS CLUSTER OPERATIONS WITH eksctl  

A) NodeGroups and Spot Instances 

--> eksctl get cluster 

--> aws eks --region us-west-2 describe-cluster --name stateless-web-app-cluster

--> eksctl get nodegroup --cluster EKS-Prod-cluster 

================================================================================= 

CLUSTER AUTOSCALER 
CONTINURE FROM HERE FOR UBUNTU SERVER (Check SCREEN SHOT) 
4) INSTALL AND CONFIGURE HELM Kubernetes Package Manager (just like yum and apt-get) 

# Installation of Helm package manager 
## Install latest version of Helm 
```bash 

cd  

--> mkdir helm && cd helm 

--> curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash 

======================2ND PART FROM RBRAC-for-iam-users =========================
# Providing RBAC to IAM users

## add a cluster admin

Steps:

1. create IAM user in AWS console (_k8s-cluster-admin_)
2. create access key for this user and store it locally
3. add user to configmap aws-auth
4. IMPORTANT: add user+accesskey to aws credentials file in dedicated section (profile)
   - TEST (Make sure user have no access): aws s3 ls --profile k8s-cluster-admin
5. No IAM/Console Policy

Execution: 

fetch current configmap before adding our user mapping

```bash
--> cd
--> kubectl -n kube-system get cm      (This command displays config map: The aws-auth)
--> kubectl -n kube-system get configmap aws-auth -o yaml (Display ConfigMap Configuration)
--> kubectl -n kube-system get configmap aws-auth -o yaml > aws-auth-configmap.yaml
--> ll or ls -al

```

edit the yaml file and add a "mapUsers" section
```bash

--> vi aws-auth-configmap.yaml

==Once in the file SCROLL DOWN AND EDIT THE mapUsers Section and Add the kubernetes user name as shown bellow===
--> To show the helm folder > Click on settings icon at the top left hand of Cloud9, and select 'Show Home In Favourite'
--> Open the aws-auth-configmap.yaml file and paste the bellow block of code
--> Edit the USERNAME and ARN and pass that of the k8s Admin
--> Save All
--> Copy the bellow block and paste (Please Make sure to delete the [] backets in that section)
``'bash
  mapUsers: |
    - userarn: K8S_ADMIN_USER_ARN
      username: K8S_ADMIN_USER_NAME
      groups:
        - system:masters
```

````
bash
==RELOAD THE FILE==
--> kubectl apply -f aws-auth-configmap.yaml -n kube-system
    NOTE: Do it a couple of times if it breaks (Delete and create, delete and create and it'll work)
--> kubectl -n kube-system get cm aws-auth
--> kubectl -n kube-system describe cm aws-auth       (If you check the Users section you'll see your k8s admin user)
````
add user to ~/.aws/credentials by creating a new section so you could use the user to Authenticate toi the API Server

```bash
--> vi ~/.aws/credentials
--> Once in the file copy the bellow block and paste at the bottom and then EDIT
[your_admin_name]
aws_access_key_id=.....
aws_secret_access_key=.....
region=us-east-1
output=json
```

check which user is currently active

```bash
--> aws sts get-caller-identity
--> export AWS_PROFILE="your_admin_name"
--> aws sts get-caller-identity

==Test==
--> kubectl get nodes
--> kubectl -n kube-system get pods
```

2) ## ADD read-only user for particular namespace

```bash
--> kubectl create namespace production
--> kubectl get namespaces

```

--> create a `Production Viewer IAM user` and grab access-keys
  --> Please do not add any permissions/policies (No permission)

--> create role
  --> Go back to terminal
  --> touch role.yml
  --> vi role.yml
  --> copy and paste the bellow yaml file in the "vi role.yml"
  --> save and quite

```bash
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: production
  name: prod-viewer-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]  # can be further limited, e.g. ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch"] 

```

3) ## create rolebinding
--> touch rolebinding
--> vi rolebinding
--> Copy and Paste the bellow rolebinding yml file and paste in "vi rolebinding"
--> save and quite

```bash
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: prod-viewer-binding
  namespace: production
subjects:
- kind: User
  name: prod-viewer
  apiGroup: ""
roleRef:
  kind: Role
  name: prod-viewer-role
  apiGroup: ""
```

create role and binding

```bash
--> kubectl apply -f role.yml 
--> kubectl apply -f rolebinding.yml
```

## add user to aws-auth configmap
Get the latest version of the config map and edit it with the prod-viewer user
--> rm aws-auth-configmap.yaml
--> kubectl -n kube-system get cm      (This command displays config map: The aws-auth)
--> kubectl -n kube-system get configmap aws-auth -o yaml > aws-auth-configmap.yaml
--> vi aws-auth-configmap.yaml    (Paste the bellow entries for the new user)

```Update the ROLE ARN and NAME. Past Right Below The ADMIN USER
- userarn: arn:aws:iam::607436220640:user/prod-viewer
      username: prod-viewer
      groups:
        - prod-viewer-role

```
--> kubectl apply -f aws-auth-configmap.yaml

add user to ~/.aws/credentials file
--> vi ~/.aws/credentials

'''
[prod-viewer]
aws_access_key_id=....
aws_secret_access_key=....
region=us-east-1
output=json
```

--> aws sts get-caller-identity
--> export AWS_PROFILE="prod-viewer"
--> aws sts get-caller-identity

```bash
aws sts get-caller-identity
export AWS_PROFILE="prod-viewer"
aws sts get-caller-identity
```

==Test prod-viewer Authorization==
--> kubectl get nodes
--> kubectl -n kube-system get pods
--> kubectl run nginx --image=nginx --restart=Never

==production name space should work TEST===
--> kubectl -n production get pods

===CONFIRMED SO SWITCH BACK to ADMIN===
--> export AWS_PROFILE="clusteradmin"
--> aws sts get-caller-identity
--> kubectl run nginx --image=nginx --restart=Never -n production

==-SWITCH BACK TO ProdViewer and TEST Again===
--> export AWS_PROFILE="prod-viewer"
--> kubectl -n production get pods

===Switch Back To k8s Admin===
--> export AWS_PROFILE="clusteradmin"

==================3RD SECTION SAMPLE-STATELESS-APP FOLDER/deploy-sample-app.md==============
# Setup sample guestbook app
* Kubernetes Repo: https://github.com/kubernetes/examples/tree/master/guestbook
* Mbandi's Repo: https://github.com/awanmbandi/deploy-stateless-app

## Redis master
deploy the master Redis pod and a _service_ on top of it:
```
--> kubectl apply -f redis-master.yaml
--> kubectl get pods
--> kubectl get services redis-master
```

## Redis slaves
deploy the Redis slave pods and a _service_ on top of it:
```
--> kubectl apply -f redis-slaves.yaml
--> kubectl get services redis-slave
--> kubectl get pods
--> kubectl get pods -o wide
--> kubectl describe node ip-192-168-28-118.ca-central-1.compute.in
```

## Frontend app
deploy the PHP Frontend pods and a _service_ of type **LoadBalancer** on top of it, to expose the loadbalanced service to the public via ELB:
```
--> kubectl apply -f frontend.yaml
```
some checks:
```
--> kubectl get pods
--> kubectl get pods -o wide
--> kubectl get service frontend
--> kubectl get pods -l app=guestbook
```
check AWS mgm console for the ELB which has been created !!!

## Access from outside the cluster
grab the public DNS of the frontend service LoadBalancer (ELB):
Please MAKE SURE TO DISCUSS THE DESCRIBE COMMAND AND HOW THE Internal and External lOAD BALANCERS. 
ALSO HOW THE PORT FORWARDING/PROXY WORKS FROM ELB TO NODES PORT 80 TO THE THE Internal ports running in the nodes.
```
--> kubectl describe service frontend

```
copy the name and paste it into your browser !!!
COPY THE LOAD BALANCER DNS ON THE CONSOLE OF VIA COMMAND LINE AND ACCESS THE APPLICATION ONM THE BROWSER

=========================4RD SECTION SCALING PODS USING kubectl=======================

## kubectl cmds for scaling pods
scaling a deployment:
```
kubectl scale --replicas <number-of-replicas> deployment <name-of-deployment>
--> kubectl get pods 
--> kubectl scale --replicas 5 deployment frontend
--> kubectl get deployment frontend
--> kubectl get pods 

You can as well scale your deployments by editing the number of pods in the frontend.yaml deployment file

```
--> kubectl get deployment redis-backend
--> kubectl scale --replicas 6 deployment redis-backend

## SCALE DOWN
--> kubectl scale --replicas 3 deployment frontend
--> kubectl get pods 

======================THE END=========================

Please clean up everything so you don't incure charges 
- Delete both CloudFormation stacks created by eksctl "Cluster Stack" and "NodeGroup"
------------------------------------------------------