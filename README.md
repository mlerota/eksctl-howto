## How to install EKS cluster with AWS TargetGroupBinding, ALB, NLB, External secrets and ArgoCD
Some of us need an option to use existing AWS loadbalancers on AWS. Some of the reasons are:
- Separate infrastructure creation from K8S
- Use one ALB and many TargetGroups (lover costs) # Although this can be done as well in K8S
- Run apps on Classic EC2 instances and K8S cluster at the same time (complex migration from classic EC2 instances to Kubernetes)
- Inability to change DNS records easily (yes there are some organizations who are not very agile and bureaucratic)

### Reguirements
- Admin access to AWS Cloud
- AWS cli
- eksctl cli
- Security group for accessing managed nodes. (Existing ALB to be able to connect to nodes)  
- Optional - VPC with 6 subnets. 3 public and 3 private

If you don't create/select VPC and your subnets, eksctl will create default ones for you. If you have many accounts and VPC's this is not an option. It can create conflict with your existing network. Proper IP addressing plan of your organization should be created in advance if you plan to have many services and environments. Also make sure you edit public subnets after creation and enable "Enable auto-assign public IPv4 address". Otherwise eksctl will fail on creation. We are also going to use Fargate profile. And for Fargate to work, you need private subnets that don't have IGW attached as a route. For Fargate pods to be able to go outside, you will need to attach NAT gateway on these private subnets.

### Steps
##### 1. TargetGroupBinding - Create security group that will enable existing AWS ELB's to access k8s nodes.

If you use standard/automatic AWS loadbalancer creation in K8S, this SG is not needed. eksctl already has everything covered. But since we want to use existing ELB's with TargetGroupBinding, SG needs to be added. Add something like allow all traffic from SG of your existing ELB that you want to use.

Then in eksctl.conf check this part:
```
    ssh:
      allow: true
      publicKeyName: <your-key-here>
      sourceSecurityGroupIds:
         - <replace-with-SG-id-from-step-1>
```
This is where we put this security group. I know it's confusing putting it in SSH part. But that's the only place I found I can add this SG for ELB's to work. I mean I can add it manually later to the nodes, but I wanted to be automatic on eksctl creation. If you want SSH access as well to the nodes, add SSH access to the same SG.

##### 2. Create a key pair for SSH access
And put the keypair name in ```publicKeyName``` field.

##### 3. Replace vaulues in eksct.conf to suit your needs

##### 4. Run the command and wait for results
You can first run test to see if you have any errors like this:<br/>
```
eksctl create cluster --config-file ./eksctl.config --dry-run
```
Sometimes you won't see all the errors with --dry-run. You will see them after 20min or so on cluster creation. When you are ready execute:
```
eksctl create cluster --config-file ./eksctl.config --profile=your-aws-profile
```
When cluster creation is finished, try to access the K8S cluster with
```
kubectl get pods -n kube-system
```
### External secrets add-on to Kubernetes
Why external secrets add-on? (https://github.com/external-secrets/external-secrets) For three simple reasons. We need something that will work on GitOps principle (passwords defined in git with the rest  of the code). Second to have passwords stored externally from Kubernetes itself. Using some managed password store like Hashicorp Vault or in our case AWS secrets manager. If we destroy the cluster,  another one can take over the same config and it will work. No manual intervention or injection of secrets in K8S cluster after deploy. And the third, we don't want to give AWS credentials or roles to k8s nodes or pods. In that way any node or pod launched in the cluster could get the secret from AWS secrets store with aws cli command. Instead we need some overlay that will inject the secret to pods upon creation. And for that we will use ArgoCD.    

### ArgoCD and External secrets
As we mentioned earlier, we want to use GitOps principle in our deployments and infrastructure. ArgoCD is great tool for that. Let's install Argo.

#### Steps

1. We need to create user `argocd` and policies that we will attach to that user. First modify files aws-secrets-m-policy.yaml and aws-ssm-policy.yaml . The files are self explanatory. Policy allow only recovering secrets and data from aws parameter store that starts with /argo/* and /k8s-cluster1/*.
```
aws iam create-policy --policy-name ArgoGetSecretsPolicy --policy-document file://aws-secrets-m-policy.yaml --profile=your-aws-profile
```
Save the output of the command. Then the same for AWS Systems Manager parameter store
```
aws iam create-policy --policy-name ArgoSSMGetPolicy --policy-document file://aws-ssm-policy.yaml --profile=your-aws-profile
```
Also save the output of the command. You will need the `arn` of the policy next.
Now create user:
```
aws iam create-user --user-name argocd --profile=your-aws-profile
```
Now attach these policies to the user you just created. Check policy arn's you saved.
```
aws iam attach-user-policy --user-name argocd --policy-arn arn:aws:iam::1234567890XX:policy/ArgoGetSecretsPolicy --profile=your-aws-profile
aws iam attach-user-policy --user-name argocd --policy-arn arn:aws:iam::1234567890XX:policy/ArgoSSMGetPolicy --profile=your-aws-profile
```
If you are going to use AWS codecommit service add this policy as well.
```
aws iam attach-user-policy --user-name argocd --policy-arn arn:aws:iam::aws:policy/AWSCodeCommitPowerUser --profile=your-aws-profile
```
Create access key and save the output
```
aws iam create-access-key --user-name argocd --profile=your-aws-profile
```
### External secrets install
```
kubectl create namespace external-secrets
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets -n external-secrets \
 external-secrets/external-secrets
```
Create keys  for AWS credentials
```
echo -n 'KEYID' > ./access-key
echo -n 'SECRETKEY' > ./secret-access-key
```
For every namespace you want to use external secrets, you need to add AWS credentials in that namespace.
```
kubectl create secret generic awssm-secret -n argocd --from-file=./access-key  --from-file=./secret-access-key
kubectl create secret generic awssm-secret -n nginx-test --from-file=./access-key  --from-file=./secret-access-key
```
More on install and external secrets here: https://external-secrets.io
#### AWS Load Balancer Controller Installation
In order to use AWS Application and Network load balancers, and TargetGroupBinding feature, we need to install this AWS Load Balancer controller. Simple old Classic Load Balancer will work out of the box as a service, and no special controllers are needed. First we need to Install the TargetGroupBinding CRDs.
```
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```
Helm part. Don't forget to change `cluster-name` in command below.
```
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
#### ArgoCD installation
There are two ways how you can install ArgoCD. With AWS SSL certificate and classic loadbalancer as a service. Or default install without SSL. Check official documentation for that. We are going to use classic AWS loadbalancer and AWS SSL certificate to access ArgoCD web UI. Of course you need to have your SSL certificate added in AWS cert manager. You can check `argo-install-v2.2.5-example.yaml` file for complete example.
```
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
# In kind: Deployment add --insecure
      containers:
      - command:
        - argocd-server
        - --insecure
```
```
# In kind: Service
name: argocd-server
   annotations:
      service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:eu-west-1:1234567890XX:certificate/343d3cd1-4321-765b-8b88-b5d8624f6c7a"
```
```    
spec:
  type: LoadBalancer
  loadBalancerSourceRanges:
    - "your.ip.add.ress/32"
    - "your.ip.net.range/24"
```
If you don't want to limit access to your LB, omit loadBalancerSourceRanges part.
Then install argocd
```
# Create namespace argocd.
kubectl create namespace argocd
# Install ArgoCD
kubectl apply -n argocd -f install.yaml
kubectl config set-context --current --namespace=argocd
```
Check the new Classic LB that is created on AWS and create DNS record to point to this LB.
Install [argocd cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/) on your PC. Then login and change default password. First get the default pass by:
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Login and change the default password. Username is admin.
```
argocd login argocd.your-hostname.com
argocd account update-password
```
To be able to use AWS codecommit for ArgoCD, we need to create SSH key for the user argocd. We will use simple command in Linux shell called `ssh-keygen`. When asked for password, just hit enter.
```
ssh-keygen                                                                      
Generating public/private rsa key pair.
Enter file in which to save the key (/home/username/.ssh/id_rsa): /home/username/.ssh/id_rsa-argo
```
Now go to AWS IAM user argocd and upload the public key you've just created. Under tab security credentials and under SSH keys for AWS CodeCommit click on Upload SSH public key.
 You will get some SSH key ID when done. Save it.
 #### Launch sample app with ArgoCD
 So far we installed everything what is needed for our project. We want to use:
 - ArgoCD as GitOps tool
 - AWS CodeCommit as code repository. (You can use Github, Gitlab etc...)
 - External secrets as secrets manager for our Kubernetes
 - Service app through existing LoadBalancer with  TargetGroupBinding (later we will add an example of 'normal' LB usage)


 1. Let's first create secret on AWS secrets manager.
 ```
 aws secretsmanager create-secret --name /argo/test1 --secret-string "This is my test secret" --profile=your-aws-profile --region=eu-west-1
 ```

2. In AWS CodeCommit create repo called argocd-test.

  #### TargetGroupBinding part
3. Create test Target Group on AWS. For target type use IP addresses, protocol http port 80 and select VPC where Kubernetes is running. Don't include any IP addresses. Copy the arn of the TargetGroup you've just created.

4. Edit the`repoURL` line from `sample-app/argocd-app-manifest.yaml` file and line `targetGroupARN:` from `sample-app/apps/example1/targetGroupBinding.yaml` file. Rename or clone/copy example1 directory to nginx-test. (refered in argocd-app-manifest.yaml file)

5. Modify your existing ALB settings - "View/edit rules" and add rules that will point to your target group we created in step 3.

6. Clone the empty argocd-test repo from CodeCommit, add the files from sample-app directory and push the changes to master.

7. Add ssh-keys to Argo step1
```
ssh-keyscan git-codecommit.<region>.amazonaws.com | argocd cert add-ssh --batch
```
8. Add the AWS CodeCommit repo to ArgoCD.
```
argocd repo add ssh://<YOUR_SSH_KEY_ID>@git-codecommit.<region>.amazonaws.com/v1/repos/argocd-test --ssh-private-key-path /home/<user>/.ssh/id_rsa-argo
```
9. Add sample app to ArgoCD
```
kubectl apply -f argocd-app-manifest.yaml
```
You can check if the external secrets is working by checking the secret by
```
kubectl get secrets nginx-secret -n nginx-test -o jsonpath="{.data.secret\.txt}" | base64 -d
```
If you don't see the secret check if you did
```
kubectl create secret generic awssm-secret -n nginx-test --from-file=./access-key  --from-file=./secret-access-key
```

10. Check the TargetGroup on AWS and under Registered targets you should see IP address that is running your deployment. If the status is unhealthy, probably you forgot to allow incoming traffic from your loadbalancer in step1. Go to the security group of kubernetes node and add the correct SG on incoming traffic.

11. External secrets. You can see the secret in pod shell in /etc/k8s-secret-nginx directory.

12. If you want to use classic loadbalancer without TargetGroupBinding, you can find the service example in example2 directory. Also Application loadbalancer example in example3 directory.

13. If you want to use ECR from another AWS account, check file ecr-from-another-acc-policy.yaml. Go to the account where you have ECR, select some repository and click on "Permissions".  Click on "Edit Policy JSON" and add policy from ecr-from-another-acc-policy.yaml. Change arn to suit the role of the nodegroup role of the k8s instance.  Add the same for every repository you want to use.
