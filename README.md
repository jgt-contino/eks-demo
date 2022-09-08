# EKS Demo

## Installation / Requirements

```bash
brew install awscli  # make sure you get v2
brew install eksctl
brew install kubectl
brew install helm # install Helm 3 client
```

Requirements:

* Admin level account created in IAM
* Profile (credentials & config files) setup in .aws folder
* Test profile access with a simple aws cli command (aws s3 ls)

## Simple

```bash
eksctl create cluster -f simple-cluster.yaml
# wait 30 minutes
kubectl get nodes
# clean up
eksctl delete cluster -f simple-cluster.yaml
```

## Add ALB Ingress Controller

```bash
# Account ID: 297041898255

# verify
eksctl create cluster -f cluster.yaml --dry-run

# create cluster
eksctl create cluster -f cluster.yaml

# add OIDC provider
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eks-test \
    --approve

# policy doc now in the repo
curl -o policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.3/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://policy.json

eksctl create iamserviceaccount \
  --cluster eks-test \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::297041898255:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve

kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

kubectl get crd

helm repo add eks https://aws.github.io/eks-charts

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eks-test --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

kubectl -n kube-system rollout status deployment aws-load-balancer-controller
```

## More Advanced (Work in Progress)

```bash
export LBC_CHART_VERSION="1.4.1"
export LBC_VERSION="v2.4.3"

# edit custom-cluster.yaml
code custom-cluster.yaml
# verify YAML config
eksctl create cluster -f custom-cluster.yaml --dry-run
# if no errors, run without dry-run option
eksctl create cluster -f custom-cluster.yaml
# go get some coffee.

# ALB Ingress - more complicated than it should be
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

# modify this command before running - change clusterName accordingly
helm upgrade -i aws-load-balancer-controller \
    eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=jgt-eks \
    --set serviceAccount.create=false \
    --set serviceAccount.name=jgt-eks \
    --set image.tag="${LBC_VERSION}" \
    --version="${LBC_CHART_VERSION}"

eksctl create iamserviceaccount \
  --cluster eksworkshop-eksctl \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
```

