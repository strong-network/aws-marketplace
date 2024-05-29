# Prerequisites
1. An EKS Cluster version 1..27 or higher.
2. An ingress controller to enable access from the outside.
3. An Elastic application loadbalancer to attach to the ingress controller.
4. A Fully Qualified Domain Name(FQDN) and all of its sub-domains that can be used to access the Strong Network Platform from the outside.
5. A cert manager to automatically issue certificates that are valid for the FQDN mentioned above
6. An ECR registry to host the Strong Network images

- Kubectl installed
- The AWS cli
- Eksctl installed
- helm version v3 or greater


# Setting up your cluster

## Deploying the helm charts
Create The appropriate namespace for the Strong Network platform deployment and resources.

```
kubectl create namespace strong
     
eksctl create iamserviceaccount \
    --name strong-service-account \
    --namespace strong \
    --cluster <ENTER_YOUR_CLUSTER_NAME_HERE> \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringFullAccess \
    --attach-policy-arn arn:aws:iam::aws:policy/AWSMarketplaceMeteringRegisterUsage \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AWSLicenseManagerConsumptionPolicy \
    --approve \
    --override-existing-serviceaccounts
```

Pull the helm charts to deploy the Strong Network platform
```
export HELM_EXPERIMENTAL_OCI=1

aws ecr get-login-password \
    --region us-east-1 | helm registry login \
    --username AWS \
    --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com

mkdir awsmp-chart && cd awsmp-chart

helm pull oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/sncommunity/helm_charts --version 1.1.1

tar xf $(pwd)/* && find $(pwd) -maxdepth 1 -type f -delete
```

Fill in the following values and deploy the helm charts to the EKS cluster

- The hostName variable must match the FQDN mentioned above
- The userAdminEmail is the username/email used to access the admin account for the Strong Network Platform
- The userAdminPassword is the password used to access the admin account for the Strong Network Platform
- The mongodb.auth.rootPassword is the password used to access the mongodb instance used by the Strong Network Platform
- The twoFaDisabled disables or enables 2 factor authentication to access the Strong Network Platform

```
helm update strong \
    --namespace strong ./* \
    --set ninja.hostName=strong.network \
    --set ninja.userAdminEmail=admin@strong.network \
    --set ninja.userAdminPassword=StrongNetworkAdminPassword1234! \
    --set ninja.mongodb.auth.rootPassword=mongodbRootPassword1234! \
    --set ninja.twoFaDisabled=true 
```
## Push the Strong Network Images to ECR

To pull the Strong Network images, run the following script:

```
aws ecr get-login-password \
    --region us-east-1 | docker login \
    --username AWS \
    --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com

AWSMP_IMAGES="709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/pycharm_python:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/intellij_ultimate:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/sncommunity/browser_in_browser:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/phpstorm_php:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/intellij_java:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/sncommunity/sn_enterprise_bundle:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/cloud_editor_generic:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/android_studio:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/sncommunity/cloud_editor_sidecar_proxy:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/sncommunity/frontend:1.0.0,709825985650.dkr.ecr.us-east-1.amazonaws.com/strong-network/workspaces/goland_go:1.0.0"
    
for i in $(echo $AWSMP_IMAGES | sed "s/,/ /g"); do docker pull $i; done
```

After pulling the Strong Network images, run the following script:

```
aws ecr get-login-password \
    --region us-east-1 | docker login \
    --username AWS \
    --password-stdin 709825985650.dkr.ecr.us-east-1.amazonaws.com