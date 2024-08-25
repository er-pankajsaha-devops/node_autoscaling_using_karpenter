# ✅ Node Autoscaling using KARPENTER.

## Overview
Optimize Kubernetes cluster resource utilization dynamically using Karpenter. 
Automatically scale nodes based on demand, ensuring efficient resource allocation.

## Concepts.

Karpenter is an open-source node lifecycle management project built for Kubernetes. Adding Karpenter to a Kubernetes cluster can dramatically improve the efficiency and cost of running workloads on that cluster. Karpenter works by:

* __Watching__ for pods that the Kubernetes scheduler has marked as unschedulable
* __Evaluating__ scheduling constraints (resource requests, nodeselectors, affinities, tolerations, and topology spread constraints) requested by the pods
* __Provisioning__ nodes that meet the requirements of the pods
* __Disrupting__ the nodes when the nodes are no longer needed

