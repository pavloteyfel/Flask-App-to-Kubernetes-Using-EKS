# Run the app at scale on AWS Cloud
In the next section of the project, you will run the app at scale on the AWS cloud. This part aims to create a CI/CD pipeline (using AWS Codebuild + CodePipeline). The steps you will follow are:

## Create an EKS Cluster and IAM Role
Check prerequisite
You should have AWS CLI, EKSCTL, KUBECTL configured in your terminal.

## Create EKS Cluster and an IAM role
You will start with creating an EKS cluster in your preferred region, using AWS CLI. Then, you will create an IAM role that the pipeline will assume to access your k8s/EKS cluster. This IAM role will have the necessary access permissions (attached JSON policies), and you will also have to add this role to the k8s cluster's configMap.

### Execution
1. Create an EKS cluster and a nodegroup containing two m5.large nodes by default:
`eksctl create cluster --name simple-jwt-api --profile <profile>`

2. Update the kube's config file:
`aws eks --region <region-code> update-kubeconfig --name <cluster_name>`

3. Check the nodes after the cluster is created:
`kubectl get nodes`

4. get aws account id:
`aws sts get-caller-identity --query Account --output text`

5. Create trust.json file:
```json
{
 "Version": "2012-10-17",
 "Statement": [
     {
         "Effect": "Allow",
         "Principal": {
             "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
         },
         "Action": "sts:AssumeRole"
     }
 ]
}
```

6. Create role:
`aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'`

7. Create policy document iam-role-policy.json:
```json
{
 "Version": "2012-10-17",
 "Statement":[{
     "Effect": "Allow",
     "Action": ["eks:Describe*", "ssm:GetParameters"],
     "Resource":"*"
 }]
}
```

8. Attach the policy to the role created:
`aws iam put-role-policy --role-name UdacityFlaskDeployCBKubectlRole --policy-name eks-describe --policy-document file://iam-role-policy.json`

9. Edig the configmap:
`kubectl get -n kube-system configmap/aws-auth -o yaml > aws-auth-patch.yml`
10. Add this to the data / mapRoles with account id:
```yaml
mapRoles: |
 - groups:
   - system:masters
   rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
   username: build      
```
11. Update the clusters config map (only works with gitbash):
```shell
 kubectl patch configmap/aws-auth -n kube-system --patch "$(cat aws-auth-patch.yml)"
```


## Deployment to Kubernetes using CodePipeline and CodeBuild
Generate a Github access token
Next, you will generate an access-token from your Github account so that whichever service has that token can access the repositories from your Github account. You will share this token with the AWS Codebuild service (programmatically) so that it can build and test your code.

### Execution
1. Prepare github access token
2. Prepare CloudFormation template ci-cd-codepipeline.cfn.yml and edit it:
- EksClusterName	Default:    name of the EKS cluster you created
- GitSourceRepo	Defaul:         github repo name
- GitBranch	Default:	        master, or any other you want to link to the Pipeline
- GitHubUser	Default:        your Github username
- KubectlRoleName	Default:    we created this role earlier
3. Upload the template in CoudFormation / Create stack / new resourse / Template is ready / Upload template
4. Specify stack details

## Create a pipeline using CloudFormation template
Create a pipeline watching for commits to your Github repository. You will create the necessary AWS resources using a script, Cloudformation template (.yaml) file, available to you. These resources collectively are called stack. It will automatically create the Codebuild and Codepipeline projects for you.

## Set a Secret using AWS Parameter Store
In order to pass your JWT secret to the app in Kubernetes securely, you will set the JWT secret using AWS parameter store.

## Build and deploy
Finally, you will trigger the manual build (on Codebuild web console) to deploy and run the app on the K8s cluster. Besides, any GitHub check-ins will also trigger the pipeline.




After project is over:
`eksctl delete cluster simple-jwt-api  --region=us-east-2`