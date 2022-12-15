# Authorization_EKS_With_IAM_Users

## Step-1: Create an IAM user 
* Create eks-test-user-1(provide any custom name) in IAM. 
 >This user will not need access to the AWS console, but programmatic access will be necessary.
 >No need to set any permissions in the creation process. No tags. Create the eks-test-user-1 and save the credentials and IAM arn.

## Step-2: Edit the aws-auth configmap in the cluster and add the "mapUsers" section.

### Verify whether aws-auth is present or not

      kubectl -n kube-system get configmap aws-auth
      
   > If present , follow the below steps
### Use the command to edit the aws-auth and add mapUsers section in the mentioned format

     kubectl -n kube-system edit configmap aws-auth
     
```
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::463907465619:role/eksctl-nv-cluster-nodegroup-nv-ng-NodeInstanceRole-N6EL5PKV51ES
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::463907465619:user/eks-test-user-1
      username: eks-test-user-1
      groups:
        - system:masters
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system

```
> Note: Now eks-test-user-1 will have the admin access as we have added the user to system defined group masters.

## Step-3:Test our new IAM user whether able to access the cluster or not

### Configure the credentials of eks-test-user-1
      aws configure --profile eks-test-user-1
>To switch between different profiles,use the below command:

      export AWS_DEFAULT_PROFILE="eks-test-user-1"(selecting current profile)
      
>To verify the user,

      aws sts get-caller-identity
>Run commands whether the user "eks-test-user-1" is able to access or not

        kubectl get nodes
        kubectl -n kube-system get pods

> Now, We have an IAM user with no permissions in our AWS account, but with admin rights in our Kubernetes cluster. Pretty cool.


### If aws-auth is not present in cluster , use below steps

     curl -o aws-auth-cm.yaml https://s3.us-east-1.amazonaws.com/amazon-eks/cloudformation/2020-10-29/aws-auth-cm.yaml

>Open the file with a text editor and Replace <ARN of instance role (not instance profile)> with the Amazon Resource Name (ARN) of the IAM role associated with your nodes, and save the file.

#### If you don't have access to s3

>create aws-auth.yaml and paste the below content

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::463907465619:role/eksctl-nv-cluster-nodegroup-nv-ng-NodeInstanceRole-N6EL5PKV51ES
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes

```
> Replace role arn with you node role arn 

### To apply, use the below command 
       kubectl apply -f aws-auth.yaml
       
### Verify aws-auth created or not
     kubectl get cm aws-auth -n kube-system
     
     
## To grant IAM user with limited access to resources of a particular namesepace , please follow the below steps

### Create an IAM user
* Create ravi(provide any custom name) in IAM using AWS console. 
 >This user will not need access to the AWS console, but programmatic access will be necessary.
 >No need to set any permissions in the creation process. No tags. Create ravi and save the credentials and IAM arn.

### Create new namespace dev in cluster

      kubectl create namespace dev

### Create role and rolebinding 

> role.yaml

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: dev
  name: deployment-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

```

> To apply ,

      kubectl apply -f role.yaml


>  role-binding.yaml

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: deployment-role-binding
  namespace: dev
subjects:
- kind: User
  name: ravi
  apiGroup: ""
roleRef:
  kind: Role
  name: deployment-role
  apiGroup: ""

```

> To apply ,
      
      kubectl apply -f role-binding.yaml
      
> Take a note of role name

### Edit the aws-auth

      kubectl edit cm aws-auth -n kube-system -o yaml
      
```
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::463907465619:role/eksctl-nv-cluster-nodegroup-nv-ng-NodeInstanceRole-N6EL5PKV51ES
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::463907465619:user/eks-test-user-1
      username: eks-test-user-1
      groups:
        - system:masters  
    - userarn: arn:aws:iam::463907465619:user/ravi
      username: ravi
      groups:
        - deployment-role
    
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"mapRoles":"- rolearn: arn:aws:iam::463907465619:role/eksctl-nv-cluster-nodegroup-nv-ng-NodeInstanceRole-N6EL5PKV51ES\n  username: system:node:{{EC2PrivateDNSName}}\n  groups:\n    - system:bootstrappers\n    - system:nodes\n"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"aws-auth","namespace":"kube-system"}}
  creationTimestamp: "2022-10-11T12:12:06Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "6664"
  uid: 1deb7d09-1dc2-4030-a942-f6c6bf1bc699


```
### Test new user ravi

      aws configure --profile ravi
      export AWS_DEFAULT_PROFILE="ravi"
      aws sts get-caller-identity

>Now user ravi able to do list of actions that we mentioned in deployment-role

### To verify , use the below command:

    kubectl get pods -n dev




