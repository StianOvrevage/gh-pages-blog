---
title: "Kubernetes Sidecar Config Drift"
description: "When using a Sidecar Injector (such as Istio), there is nothing that ensures that an update (potentially breaking) to a sidecar config template is applied/updated on Pods that have already been injected with a sidecar. This post describes the causes of this problem, as well as introducing a tool to mitigate it."
image: 2022-05-03-kubernetes-sidecar-config-drift.png
date: 2022-05-03 00:00:00 +0000
categories:
    - technology
    - tools
tags:
    - kubernetes
excerpt_separator: <!--more-->
---

*This is a cross-post of a blog post also published on the [Signicat Blog](https://www.signicat.com/blog/kubernetes-your-sidecar-configurations-are-drifting)*

---

Huge thanks to one of my favorite clients, [Signicat](https://www.signicat.com/), and especially [Jon](https://www.linkedin.com/in/jon-skarpeteig/), for allowing me to share some of the nitty gritty details of a challenge that I believe is probably quite widespread, yet under-appreciated, in modern Kubernetes cloud environments.

Last week, working on Signicat’s next generation cloud platform, I discovered that several individuals invented their own ways of mitigating what I now call Sidecar Configuration Drift.

To ease the pain I created [k8s-sidecar-rollout](https://github.com/StianOvrevage/k8s-sidecar-rollout) to restart the required workloads and thereby updating their sidecar configurations.

This blog post is a bit of background information on what the causes of this problem is.

## What is Sidecar Config Drift?

When using a Sidecar Injector (such as Istio), there is nothing that ensures that an update (potentially breaking) to a sidecar config template is applied/updated on Pods that have already been injected with a sidecar.

This means that after updating a sidecar config it may take a very long time until all Pods have the updated config. They may receive the updates at any undetermined time in the future. While the updates are pending things might not work as expected and things may not be compliant. When the update is finally applied to the Pod it may surface breaking changes.

I call this phenomena _Sidecar Configuration Drift_.

## Background

### Kubernetes - Declarative vs imperative

What makes Kubernetes so powerful is also what can make it hard and confusing to work with until your mindset has shifted.

That is the philosophy of being declarative instead of imperative like most of us are used to for the past 20 years.

__Declarative means you tell Kubernetes HOW you want things to look. You DON’T tell Kubernetes WHAT to do.__

For example you do not tell Kubernetes to scale your Deployment to 5 instances. You tell Kubernetes that your Deployment should have 5 instances. The difference is subtle but extremely important. In a highly dynamic cloud environment your instances might disappear or crash for a multitude of reasons. In this declarative mindset you should not care about that since Kubernetes is tasked with ensuring 5 instances and will (try to) provision new ones when it’s needed.

This is the apparent magic which makes Kubernetes.

This magic is technically solved with what we call __Controllers__. A controller is responsible for constantly comparing the __Actual state__ and __Desired state__. To scale your Deployment to 5 instances you set `replicas: 5` on the `Deployment` resource. If needed a controller will create a completely new `ReplicaSet` resource with 5 instances. Another controller will then create 5 `Pods`. And the scheduler will finally try to place and start those `Pods` on actual nodes. The `ReplicaSet` controller will scale down the old once the new has reached it’s desired state.

This constant process is called a __reconciliation loop__ and it’s a critical feature.

### Sidecars

Sidecar is a Kubernetes design pattern. A sidecar is simply a container that is living side-by-side with the main application container that can do tasks that are logically not part of the application. Examples of this can be log handling, monitoring agents, proxies. __All containers in a Pod (including sidecars) share the same filesystem, kernel namespace, IP addresses etc.__

More about sidecars: [The Distributed System ToolKit: Patterns for Composite Containers](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns/)

### Sidecar injection

Sidecar injection is when a sidecar container is added to a Pod __even if the sidecar isn’t defined in any of the higher level primitives, such as `Deployment` or `StatefulSet`.__

When for example the `ReplicaSet` controller will `Create` a new `Pod`. If configured, the Kubernetes API will call one or more `MutatingWebhooks`. These webhooks can then change the `Pod` definition before they are saved (and picked up by the scheduler).

### The Problem

The problem is when updating a sidecar injection template there is no system that runs a reconciliation loop.

The webhook just updates the `Pod` template. It does not keep track of which `Pods` have gotten which template or check if any template change would result in a different sidecar configuration.

The controllers ALSO does not continuously monitor if and how re-creating the same `Pod` (without sidecars) would result in a different Pod once the sidecars have been injected.

In effect sidecar injection does not follow the expected declarative pattern that the rest of Kubernetes does.

### Consequences

If a platform team changes the istio sidecar template it will not actually take effect on a `Pod` until that `Pod` for some reason is re-created.

Let’s assume the `istio-proxy` sidecar template have been updated by the platform team. We roll it out and test it and it seems to work. But the change will break some applications running in the cluster.

That breakage will go un-noticed until:

    The product team commits changes that triggers a re-deploy. The deployment will suddenly fail but it might not have anything to do with the actual changes the team did to the application. This is surely confusing!

    The platform team for example upgrades a pool of worker nodes causing all `Pods` to be re-created on new nodes.

    A Pod is re-created when a Kubernetes worker node crashes. In this scenario it appears the failure spawned into existence out of nowhere since neither the product team nor platform team actually “did” anything to trigger it.

__Also worth noting is that any attempts at Rolling back a Deployment now containing failing Pods will not actually fix anything.__

It’s the sidecar templates that needs to be rolled back and Pods probably need to be re-created again.

### Mitigations

We can mitigate drift by:

    Re-starting all Pods in the cluster whenever we update sidecar injection templates.

    Sometimes we might forget to re-start so regularly re-start all Pods in the cluster anyway.

## k8s-sidecar-rollout

To make these restarts easy and fast I’ve created https://github.com/StianOvrevage/k8s-sidecar-rollout .

It’s a tool that figures out (with your help) which workloads (Deployment, StatefulSet, DaemonSet) that needs to be rolled out again (re-started) and then rolls out for you. Head over to the GitHub repo for installation and complete usage instructions. Here is an example of how it can be used:

    python3 sidecar-rollout.py \
        --sidecar-container-name=istio-proxy \
        --include-daemonset=true \
        --annotation-prefix=myCompany \
        --parallel-rollouts 10 \
        --only-started-before="2022-05-01 13:00" \
        --exclude-namespace=kube-system \
        --confirm=true

This will gather all Pods with a container named `istio-sidecar` belonging to a Deployment or DaemonSet that was started before 2022-05-01 13:00 (which may be when we updated the istio sidecar config template) excluding the `kube-system` namespace. It will patch the workloads with two annotations with `myCompany` prefix and run 10 rollouts in parallel.

The script that now re-starts Pods adds two annotations indicating that a restart to update sidecars has occurred as well as the time:

    $ kubectl get pods -n product-team some-api-7cdc65482b-ged13 -o yaml | yq '.metadata.annotations'
        sidecarRollout.rollout.timestamp: 2022-05-03T18:05:31
        sidecarRollout.rollout.reason: Update sidecars istio-proxy

The idea is that if your Pods are suddenly failing, you can quickly check the annotations and see if it has anything to do with sidecar updates or not.

These annotations will of course disappear again when a Deployment is updated.

