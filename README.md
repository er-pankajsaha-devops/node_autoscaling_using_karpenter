# Karpenter: Automatic Node Scaling for Kubernetes

## Overview
Optimize Kubernetes cluster resource utilization dynamically using Karpenter.Automatically scale nodes based on demand, ensuring efficient resource allocation.

## Concepts.

Karpenter is an open-source project that automates node scaling in Kubernetes clusters. It dynamically adjusts the number of nodes based on resource demands, ensuring efficient utilization and cost savings.

* __Watching__ for pods that the Kubernetes scheduler has marked as unschedulable
* __Evaluating__ scheduling constraints (resource requests, nodeselectors, affinities, tolerations, and topology spread constraints) requested by the pods
* __Provisioning__ nodes that meet the requirements of the pods
* __Disrupting__ the nodes when the nodes are no longer needed


## Key Features

1. **Autoscaling**:
   - Karpenter automatically scales the number of nodes up or down based on pod resource requirements.
   - It ensures optimal resource allocation without manual intervention.

2. **Customizable Policies**:
   - Define scaling policies based on metrics like CPU, memory, or custom metrics.
   - Set thresholds for scaling actions (e.g., add or remove nodes when CPU utilization exceeds 80%).

3. **Spot Instance Support**:
   - Karpenter integrates with cloud providers' spot instances (e.g., AWS EC2 Spot Instances).
   - Use spot instances to reduce costs while maintaining reliability.

## Important Components

1. **Karpenter Controller**:
   - The core component responsible for monitoring resource demands and triggering scaling events.
   - Runs as a Kubernetes controller.

2. **Scaling Policies (KarpenterScalingPolicy)**:
   - Custom resources (CRs) that define scaling rules.
   - Specify thresholds, minimum, and maximum node counts.

3. **NodePools**:
   - Logical groups of nodes with similar characteristics (e.g., instance type, availability zone).
   - Karpenter manages node scaling within these pools.

4. **EC2NodeClasses**:
   - Node classes specific to AWS EC2 instances.
   - Define instance types, spot instance settings, and labels.

5. **NodeClaims**:
   - Requests from workloads for specific node characteristics (e.g., GPU, specialized hardware).
   - Karpenter ensures nodes match these claims.

## Implementation steps - [AWS-EKS cluster with Karpenter](https://github.com/er-pankajsaha-devops/node_autoscaling_using_karpenter/blob/main/Implementation-AWS/README.md)

## Conclusion

Karpenter simplifies node management, improves resource utilization, and enhances cluster efficiency. Happy autoscaling! 🚀

Official Karpenter documentation [Link](https://karpenter.sh/docs/)