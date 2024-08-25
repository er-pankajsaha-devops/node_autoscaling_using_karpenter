# Overview

__EC2NodeClasses__ allow you to customize AWS-specific settings for your Kubernetes nodes.Each NodePool can reference an EC2NodeClass, ensuring consistent configurations across your cluster.

## What Is an EC2NodeClass?

An EC2NodeClass defines various attributes for your EC2 instances, such as instance types, spot instance settings, IAM roles, and more.

## Steps to Create an EC2NodeClass

1. **Create an EC2NodeClass Custom Resource (CR)**:
   - Define an EC2NodeClass YAML file (e.g., `ec2nodeclass.yaml`).
   - Specify the desired settings, such as CPU, memory, and eviction thresholds.
   - Example EC2NodeClass:
     ```yaml
     apiVersion: karpenter.k8s.aws/v1
     kind: EC2NodeClass
     metadata:
       name: default
     spec:
       # IAM role to use for the node identity.Must specify one of "role" or "instanceProfile" for Karpenter to launch nodes
       role: "KarpenterNodeRole-${CLUSTER-NAME}"
       #instanceProfile: "KarpenterNodeInstanceProfile-${CLUSTER_NAME}"
       subnetSelectorTerms:
       # Select on any subnet that has the "karpenter.sh/discovery: ${CLUSTER_NAME}"
         - tags:
             karpenter.sh/discovery: "${CLUSTER_NAME}"
       # Required, discovers security groups to attach to instances 
       securityGroupSelectorTerms:
       # Either you can use tag labor selector, or sg name or sg id.
         - tags:
             karpenter.sh/discovery: "${CLUSTER_NAME}"
             environment: test
         - name: ${sg-name}
         - id: ${sg-id}
       # Required, resolves a default ami i.e AL2, Populate it Custom to use your own eks customised AMI.
       amiFamily: Custom
       amiSelectorTerms:
         - id: "${ami-id}"
       tags:
       # Following tags will be attached to the VMs provisioned by Karpenter.
         Env: Test
         Name: Test.${cluster-Name}
       userData:
         |
         ${Custom-User specific data}
           
     ```

2. **Apply the EC2NodeClass**:
   ```bash
   kubectl apply -f ec2nodeclass.yaml
   ```

3. **Verify Configuration**:
   Deploy workloads to observe Karpenter using the specified EC2NodeClass for autoscaling.

## Conclusion
EC2NodeClasses simplify AWS-specific node management, allowing you to fine-tune your instances based on your requirements.

Official Karpenter EC2Nodeclass - [Link](https://karpenter.sh/docs/concepts/nodeclasses/)