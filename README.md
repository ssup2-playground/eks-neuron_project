# eks-neuron Project

eks-neuron is prototyping project for testing inferentia/trainium instances based on EKS. eks-neuron project consists of the following git repositories.

* [aws-terraform](https://github.com/aws-playground/eks-neuron_aws-terraform) : Terraform for EKS cluster and 
* [inf1-serving-app](https://github.com/aws-playground/eks-neuron_inf1-serving-app) : ML model serving application based on inferentia 1 and FastAPI.

## Architecture

<img src="/images/architecture.png" width="800"/>

* "core" Managed Node Group
  * Running CoreDNS and Karpenter

* "add-on" Karpenter NodePool
  * Running prometheus server, my-scheduler and neuron-scheduler extender.
  * When multiple inferentia cores are assigned to one pod, my-scheduler and neuron-scheduler extender plays the role of allocating cores consecutively.
  * my-scheduler operates in active-standby mode and ensures high availability.
  * neuron-scheduler operates in active-active mode and ensures high availability.

* "inf" Karpenter NodePool
  * Running serving app, neuron device plugin, neuron monitor and node problem detector.
  * Neuron device plugin makes the kubelet aware of the inferentia core.
  * Neuron monitor provides inferentia/trainium metrics to prometheus server. 
  * Node problem detector detects failure inferentia/trainium cores.

## Inf1 Serving App

* based on inferentia 1 and FastAPI
* serving app uses my-scheduler to allocate mutiple inferentia cores sequentially

