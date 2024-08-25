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
    - Now we need to create an IAM role that the Karpenter controller will use to provision new instances. The controller will be using IAM Roles for Service Accounts (IRSA) which requires an OIDC endpoint.
       ```bash
       cat << EOF > controller-trust-policy.json
       {
            "Version": "2012-10-17",
            "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                "Federated": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC_ENDPOINT#*//}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${OIDC_ENDPOINT#*//}:aud": "sts.amazonaws.com",
                    "${OIDC_ENDPOINT#*//}:sub": "system:serviceaccount:${KARPENTER_NAMESPACE}:karpenter"
                    }
                }
                }
            ]       
        }
        EOF

        aws iam create-role --role-name "KarpenterControllerRole-${CLUSTER_NAME}"--assume-role-policy-document file://controller-trust-policy.json

        cat << EOF > controller-policy.json 
        {
            "Statement": [
                {
                    "Action": [
                    "ssm:GetParameter",
                    "ec2:DescribeImages",
                    "ec2:RunInstances",
                    "ec2:DescribeSubnets",
                    "ec2:DescribeSecurityGroups",
                    "ec2:DescribeLaunchTemplates",
                    "ec2:DescribeInstances",
                    "ec2:DescribeInstanceTypes",
                    "ec2:DescribeInstanceTypeOfferings",
                    "ec2:DescribeAvailabilityZones",
                    "ec2:DeleteLaunchTemplate",
                    "ec2:CreateTags",
                    "ec2:CreateLaunchTemplate",
                    "ec2:CreateFleet",
                    "ec2:DescribeSpotPriceHistory",
                    "pricing:GetProducts"
                    ],
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "Karpenter"
                },
                {
                    "Action": "ec2:TerminateInstances",
                    "Condition": {
                        "StringLike": {
                            "ec2:ResourceTag/karpenter.sh/nodepool": "*"
                        }
                    },
                    "Effect": "Allow",
                    "Resource": "*",
                    "Sid": "ConditionalEC2Termination"
                },
                {
                    "Effect": "Allow",
                    "Action": "iam:PassRole",
                    "Resource": "arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}",
                    "Sid": "PassNodeIAMRole"
                },
                {
                    "Effect": "Allow",
                    "Action": "eks:DescribeCluster",
                    "Resource": "arn:${AWS_PARTITION}:eks:${AWS_REGION}:${AWS_ACCOUNT_ID}:cluster/${CLUSTER_NAME}",
                    "Sid": "EKSClusterEndpointLookup"
                },
                {
                    "Sid": "AllowScopedInstanceProfileCreationActions",
                    "Effect": "Allow",
                    "Resource": "*",
                    "Action": [
                    "iam:CreateInstanceProfile"
                    ],
                    "Condition": {
                    "StringEquals": {
                        "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                        "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
                    },
                    "StringLike": {
                        "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
                    }
                    }
                },
                {
                    "Sid": "AllowScopedInstanceProfileTagActions",
                    "Effect": "Allow",
                    "Resource": "*",
                    "Action": [
                    "iam:TagInstanceProfile"
                    ],
                    "Condition": {
                    "StringEquals": {
                        "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                        "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}",
                        "aws:RequestTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                        "aws:RequestTag/topology.kubernetes.io/region": "${AWS_REGION}"
                    },
                    "StringLike": {
                        "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*",
                        "aws:RequestTag/karpenter.k8s.aws/ec2nodeclass": "*"
                    }
                    }
                },
                {
                    "Sid": "AllowScopedInstanceProfileActions",
                    "Effect": "Allow",
                    "Resource": "*",
                    "Action": [
                    "iam:AddRoleToInstanceProfile",
                    "iam:RemoveRoleFromInstanceProfile",
                    "iam:DeleteInstanceProfile"
                    ],
                    "Condition": {
                    "StringEquals": {
                        "aws:ResourceTag/kubernetes.io/cluster/${CLUSTER_NAME}": "owned",
                        "aws:ResourceTag/topology.kubernetes.io/region": "${AWS_REGION}"
                    },
                    "StringLike": {
                        "aws:ResourceTag/karpenter.k8s.aws/ec2nodeclass": "*"
                    }
                    }
                },
                {
                    "Sid": "AllowInstanceProfileReadActions",
                    "Effect": "Allow",
                    "Resource": "*",
                    "Action": "iam:GetInstanceProfile"
                }
            ],
            "Version": "2012-10-17"
            }
            EOF

        aws iam put-role-policy --role-name "KarpenterControllerRole-${CLUSTER_NAME}" --policy-name "KarpenterControllerPolicy-${CLUSTER_NAME}" --policy-document file://controller-policy.json

       ```
4. Add tags to subnets and security groups
   - We need to add tags to our nodegroup subnets so Karpenter will know which subnets to use.
       ```bash
       for NODEGROUP in $(aws eks list-nodegroups --cluster-name "${CLUSTER_NAME}" --query 'nodegroups' --output text); 
       do
            aws ec2 create-tags --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
            --resources $(aws eks describe-nodegroup --cluster-name "${CLUSTER_NAME}" \
            --nodegroup-name "${NODEGROUP}" --query 'nodegroup.subnets' --output text )
        done
       ```
    - Add tags to our security groups. This command only tags the security groups for the first nodegroup in the cluster. If you have multiple nodegroups or multiple security groups you will need to decide which one Karpenter should use
        ```bash
        NODEGROUP=$(aws eks list-nodegroups --cluster-name "${CLUSTER_NAME}" \
            --query 'nodegroups[0]' --output text)

        LAUNCH_TEMPLATE=$(aws eks describe-nodegroup --cluster-name "${CLUSTER_NAME}" \
            --nodegroup-name "${NODEGROUP}" --query 'nodegroup.launchTemplate.{id:id,version:version}' \
            --output text | tr -s "\t" ",")

        # If your EKS setup is configured to use only Cluster security group, then please execute -

        SECURITY_GROUPS=$(aws eks describe-cluster \
            --name "${CLUSTER_NAME}" --query "cluster.resourcesVpcConfig.clusterSecurityGroupId" --output text)

        # If your setup uses the security groups in the Launch template of a managed node group, then :

        SECURITY_GROUPS="$(aws ec2 describe-launch-template-versions \
            --launch-template-id "${LAUNCH_TEMPLATE%,*}" --versions "${LAUNCH_TEMPLATE#*,}" \
            --query 'LaunchTemplateVersions[0].LaunchTemplateData.[NetworkInterfaces[0].Groups||SecurityGroupIds]' \
            --output text)"

        aws ec2 create-tags \
            --tags "Key=karpenter.sh/discovery,Value=${CLUSTER_NAME}" \
            --resources "${SECURITY_GROUPS}"

        ```
5. Update aws-auth ConfigMap
    - We need to allow nodes that are using the node IAM role we just created to join the cluster. To do that we have to modify the aws-auth ConfigMap in the cluster.
        ```bash
        kubectl edit configmap aws-auth -n kube-system
        ```
    - You will need to add a section to the mapRoles that looks something like this. Replace the ${AWS_PARTITION} variable with the account partition, ${AWS_ACCOUNT_ID} variable with your account ID, and ${CLUSTER_NAME} variable with the cluster name, but do not replace the {{EC2PrivateDNSName}}.
        ```bash 
        - groups:
        - system:bootstrappers
        - system:nodes
        ## If you intend to run Windows workloads, the kube-proxy group should be specified.
        # For more information, see https://github.com/aws/karpenter/issues/5099.
        # - eks:kube-proxy-windows
        rolearn: arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME}
        username: system:node:{{EC2PrivateDNSName}}
        ```
    - The full aws-auth configmap should have two groups. One for your Karpenter node role and one for your existing node group
6. Deploy Karpenter
   - First set the Karpenter release you want to deploy
        ```bash
        export KARPENTER_VERSION="1.0.1"
        ```
    - We can now generate a full Karpenter deployment yaml from the Helm chart.
        ```bash
        helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version "${KARPENTER_VERSION}" --namespace "${KARPENTER_NAMESPACE}" \
        --set "settings.clusterName=${CLUSTER_NAME}" \
        --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:role/KarpenterControllerRole-${CLUSTER_NAME}" \
        --set controller.resources.requests.cpu=1 \
        --set controller.resources.requests.memory=1Gi \
        --set controller.resources.limits.cpu=1 \
        --set controller.resources.limits.memory=1Gi > karpenter.yaml
        ```
    - Now that our deployment is ready we can create the karpenter namespace, create the NodePool CRD, and then deploy the rest of the karpenter resources.
        ```bash
        kubectl create namespace "${KARPENTER_NAMESPACE}" || true
        kubectl create -f "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.sh_nodepools.yaml"
        kubectl create -f "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.k8s.aws_ec2nodeclasses.yaml"
        kubectl create -f "https://raw.githubusercontent.com/aws/karpenter-provider-aws/v${KARPENTER_VERSION}/pkg/apis/crds/karpenter.sh_nodeclaims.yaml"
        kubectl apply -f karpenter.yaml
        ```
## NodePools, EC2nodeclasses & Nodeclaims.
