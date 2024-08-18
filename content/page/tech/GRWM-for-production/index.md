---
title: "GRWM: for production"
date: 2024-08-08T12:32:12+11:00
draft: true
topic: tech
---

# GRWM for production: be production ready

Things to ask yourself.


## What does the workload do and what does it need to run?

#### Workload Management

It's important to understand the [workload mangement](https://kubernetes.io/docs/concepts/workloads/controllers/) options Kubernetes offers and selecting the appropriate one for the application.

Generally applications should run as deployments. 
Specialised services that monitor things on nodes should be daemonsets, and certain stateful workloads that depend on cardinal services and application state need to be statefulsets.
Kubernetes is best suited for stateless workloads, and to get the most out of the "self-healing" powers and scaleability of k8s you should do your best to externalise any state.

A quick dirty flowchart:

![image](images/workload-management-flow-chart.png)


#### Scheduling
- What are the application's resource requirements? This includes CPU, memory
- Node requirements
  Are there specialised nodes (high memory, or different architectures, need GPUs?)

#### Data

#### Config


#### liefcycle
- preStop hooks




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
