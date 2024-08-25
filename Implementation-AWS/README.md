# Karpenter provider for AWS with an existing EKS cluster.

* We will use an existing EKS cluster
* You will use existing VPC and subnets
* You will use existing security groups
* Your nodes are part of one or more node groups
* Your workloads have pod disruption budgets that adhere to EKS best practices
* Your cluster has an OIDC provider for service accounts
* User must have `aws-cli` installed.

## Steps
1. Set a variable for your cluster name.
```bash
    KARPENTER_NAMESPACE=kube-system
    CLUSTER_NAME=<your cluster name>
```

2. Set other variables from your cluster configuration.
```bash
    AWS_PARTITION="aws" # if you are not using standard partitions, you may need to configure to aws-cn / aws-us-gov
    AWS_REGION="$(aws configure list | grep region | tr -s " " | cut -d" " -f3)"
    OIDC_ENDPOINT="$(aws eks describe-cluster --name "${CLUSTER_NAME}" --query "cluster.identity.oidc.issuer" --output text)"
    AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
    K8S_VERSION=1.28
```
3. Create IAM Roles - To get started with our migration we first need to create two new IAM roles for nodes provisioned with Karpenter and the Karpenter controller.
    - To create the Karpenter node role we will use the following policy and commands.
        ```bash
        echo '{
        "Version": "2012-10-17",
        "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
       ]
       }' > node-trust-policy.json

       aws iam create-role --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --assume-role-policy-document file://node-trust-policy.json
       ```
    - Now attach the required policies to the role
       ```bash
       aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKSWorkerNodePolicy"

       aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEKS_CNI_Policy"
       
       aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
       
       aws iam attach-role-policy --role-name "KarpenterNodeRole-${CLUSTER_NAME}" --policy-arn "arn:${AWS_PARTITION}:iam::aws:policy/AmazonSSMManagedInstanceCore"

       ```