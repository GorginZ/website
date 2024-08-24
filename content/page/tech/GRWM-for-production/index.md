---
title: "GRWM: for production"
date: 2024-08-08T12:32:12+11:00
draft: true
topic: tech
---

# GRWM: Get your kubernetes workload ready for production

In this piece, we will explore the key considerations for configuring a resilient workload and preparing it for production in Kubernetes. By asking ourselves the right questions, we can effectively leverage the orchestration features provided by Kubernetes to ensure our workload is well-configured and ready for deployment.



## Workload Management

#### *How* should we run it?

It's important to understand the [workload mangement](https://kubernetes.io/docs/concepts/workloads/controllers/) options Kubernetes offers and selecting the appropriate controller for the service. What the workload itself actually does is a great starting point.

Most applications should run as [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). To get the most out of the "self-healing" powers and scaleability of k8s you should do your best to externalise any application state and allow your application instances to be totally interchangeable. There may be some "pre work" to do that can help prepare the application itself for deployment - read about [The Twelve Factor App.](https://12factor.net/) 


Kubernetes does accommodate stateful workloads with [StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) and [PersistentVolumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). These can be used to leverage the ordinal pod numbers for workloads where the unique identity of the instance matters. 



Workloads that could be described as "tasks" will be suited to [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) or [Jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/) which deserve their own post on configuring.

#### A quick dirty flowchart:

![image](images/workload-management-flow-chart.png)
>Note: if you are *not* a cluster admin, disregard daemonSets in the flow-chart. Daemonsets run a pod on every node, specialised system workloads like network plugins are an example of a workload that must be run as a DaemonSet.

#### Scheduling

To get scheduled on an appropriate node by [kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) the workload should define what resources or other host requirements it needs.

- What are the application's resource requirements? This includes CPU, memory and ephemeral storage. What would appropriate requests be?
- Node requirements:
  - Are there specialised nodes available that this workload needs to use (high memory, or different architectures, need GPUs?)
  - Should the workload isolated from other workloads?
  - OS?
- Is the workload "critical" or very high priority? Administrators may allow you to leverage [PriorityClasses](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass). 


#### Application Config

What does the application need with respect to configuration?
Your application probably needs some values that it requires to run as well as other environment particular configurations that differ between deployment environments - like where to reach an external service it calls and credentials. 

There are multiple ways to expose these values to workloads by leveraging the native Kubernetes objects. What do you need to pass in to your workload? We can achieve this with:

- [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
  - designed for small sensitive data. For decoupling sensitive data from code. NOTE THAT THEY ARE STORED UNENCRYPTED IN ETCD. So often secrets are not used for *actually* sensitive data. Consider other secret management services that integrate with k8s here.
- [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
  - For non-confidential data, is in key-value pairs.
  - great for things like nginx config file, or a set of configuration key pair values that you can mount as envVars fo the application.
 - You can [mount objects like ConfigMaps as Volumes for your container to consume.](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap)  
- [Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) - You can set environment variables directly (which you could template) or by referencing an object like a Secret, ConfigMap or Volume
- If there are some special actions that need to take place for configuration that for some reason can't be served by already existing objects, or there's a timing problem that the value for something can't be known until whenever - [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) could be leveraged to do the work of helping create the configuration for the "main" app and updating a shared volume for the pod (just as an example). Will discuss these in lifecycle management section.

Read the kubernetes [workload configuration best practices](https://kubernetes.io/docs/concepts/configuration/overview/#general-configuration-tips) 

#### liefcycle

- init containers if needed.
- preStop hooks


#### Be a good citizen

- resource limits
- let cronjobs fail nicely.



# A hypothetical

In this piece I'll prepare a hypothetical app to deploy in kubernetes to show some of the important first things we should configure to get production ready that will apply broadly to most workloads.

In our scenario the developers have already containerised their app and are seeking our guidence on how to get production ready and deploy to a cluster. We're going to step them through.

## What does the workload do and what does it need to run?

For this hypothetical I'm going to say our cool app is some backend component that is talked to by a frontend in the cluster and also talks to an external database.

- It expects to run in a linux os.
- needs to be able to communicate to cool-frontend service
- needs to be able to communicate to external postgres db


#### Workload Management

The first thing we need to decide is what is appropriate [workload mangement](https://kubernetes.io/docs/concepts/workloads/controllers/) option for our service
If our workload is always meant to be up and running we'll be looking at Deployments, StatefulSets and Daemonsets. If the workload is completing a task we'd consider a job or cronjob.

Our application most suites a Deployment, we want to be able to run multiple replicas but we don't need statefulness and we don't need our workload to run on all nodes so we wouldn't even consider those as viable options in this case.

#### Let's start building out the spec

let's get a yaml to get started with:

```bash
k create deployment cool-backend --image cool-backend:v0.0.1 --dry-run=client -o yaml > cool-deployment.yaml
```

This gives us something to start with. We can remove some bits:

```diff
apiVersion: apps/v1
kind: Deployment
metadata:
--  creationTimestamp: null
  labels:
    app: cool-backend
  name: cool-backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cool-backend
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: cool-backend
    spec:
      containers:
      - image: cool-backend:v0.0.1
        name: cool-backend:w

        resources: {}
--status: {}
```

----

#### scheduling
kube-scheduler will schedule your workload best it can, ranking nodes for suitability. We can help kube-scheduler find the best node for us.

- is your workload CPU or memory bound?
- architecture?
- special nodes?

## Be a good citizen 

Multitenant clusters owns and run by a platform team are great, but we have a responsability to not be greedy! If you're running in a multi-tenant cluster it's likely there are limits in place to prevent resource hogging - also maybe there's chargeback so we should care about what we really need.


## Releases
- how are you promoting images?
- strategy?
- CICD?
- k8s side, PDB, how blue green implemented

# Be reliable
high availability

fault tolerance 

scaleability

- when do you need to scale up? (cpu, memory, custom metrics like queue length, latency)
- horizontal or vertical scaling? What is available in the cluster?

## Healthz


## Templating


#### Thoughts and opinions


Where possible it's simplest to avoid running stateful applications in Kubernetes, that could mean using a managed service instead of running your own database in kubernetes or if you're running something like Prometheus, moving to a scrape and forward model so the workload itself could be as close to stateless as you can get (and allow you to run it as a Deployment or Daemonset instead of a StatefulSet)
