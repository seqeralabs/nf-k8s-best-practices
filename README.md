# Nextflow on Kubernetes: Best Practices

**ARCHIVED**: This guide is no longer maintained. Instead, refer to the [Nextflow documentation](https://nextflow.io/docs/latest/kubernetes.html) as well as our Kubernetes-related blog posts:
- [The State of Kubernetes in Nextflow](https://nextflow.io/blog/2023/the-state-of-kubernetes-in-nextflow.html)
- [Nextflow and K8s Rebooted: Running Nextflow on Amazon EKS](https://seqera.io/blog/deploying-nextflow-on-amazon-eks/)

## Introduction

This document is a collaborative space for users and developers to contribute accumulated knowledge from running Nextflow on Kubernetes. Anyone is welcome to add content as they see fit and I (@bentsherman) will try to keep things organized as the project develops.

## When to Use Kubernetes

Kubernetes is a container orchestration system that can run atop on-prem hardware or in the cloud. In some cases, Kubernetes may only add an unnecessary layer of abstraction. For example, to run Nextflow on AWS, you can either provision an EKS cluster and run Nextflow with Kubernetes, or you can use AWS Batch directly. Both approaches support containers, so the EKS cluster may only introduce more overhead without much benefit. On the other hand, if you have an on-prem system with no cloud management layer (e.g. OpenStack), then you might consider using Kubernetes instead of a traditional HPC scheduler such as SLURM.

In the case of cloud-based Kubernetes, here are some other questions to consider:

- Do you want to use any Kubernetes-specific features (e.g. auto-scaling library or a special type of storage) ?
- Does your workflow execute many short-running tasks or a few long-running tasks? The cloud batch executors are not as efficient in the former case.
- Do you want VMs to be provisioned on the fly (cloud batch) or provision a fixed set of nodes that run pods in parallel (Kubernetes)?

## Summary of Nextflow Documentation

Refer to the [Nextflow documentation](https://www.nextflow.io/docs/latest/kubernetes.html) for information on how to use Nextflow with Kubernetes. The summary of the documentation is as follows:

- Pipeline should be available via Git repository (e.g. GitHub) and containerized via Docker registry (e.g. DockerHub)
- Kubernetes cluster should have a ReadWriteMany PVC and a service account with “view” and “edit” roles
- To run a pipeline: (1) create a submitter pod, (2) fetch Nextflow pipeline, (3) launch pipeline with k8s executor and k8s PVC settings
- Nextflow kuberun automates the above process, but is an experimental feature
- For non-trivial deployments, submitter pod should be managed by an external service (e.g. Nextflow Tower) instead of kuberun
- The pod directive allows a subset of config objects to be passed to container (environment variables, volume claims, node selector, etc)

## Tips, Tricks, and Caveats

### Preparing your pipeline for Kubernetes

You will most likely not need to add any k8s-specific settings to your pipeline to make it work with k8s. Most of the work is just publishing and containerizing your pipeline, which is a good practice on its own. Preparing for k8s is more about configuring your k8s cluster to work with Nextflow, that is, providing a PVC and a service account with appropriate privileges. Check out the [nf-tower-k8s](https://github.com/seqeralabs/nf-tower-k8s) repo for more guidance on cluster preparation. Although it is primarily for Tower users, it covers the same prerequisites for running Nextflow on K8s.

### Accessing your PVC

Nextflow and all of its tasks store everything on your PVC. You can use `nextflow kuberun login -v <pvc>` to browse your PVC for debugging purposes. This command is simply a shortcut for creating a pod with the PVC attached. Alternatively, you can just exec into any running workflow pod, for example `kubectl exec -it <pod-name> –- bash`.

### Using kuberun

Launching pipelines with `kuberun` is convenient, but __it is experimental and should only be used for testing__. Here are just a few known issues and limitations with `kuberun` (at the time of writing):
- Implicit variables like `$baseDir` and function definitions are not recognized, even if defined in the repository's config file (nextflow-io/nextflow#1050)
- Some config settings may not be applied, e.g. `k8s.serviceAccount` (nextflow-io/nextflow#2266)
- Pipelines must be available from a Git repository, local pipelines cannot be tested

In cases where `kuberun` doesn’t work for you, you can always get around it by either (1) creating your own workflow pod (like a head node on a HPC system) or (2) using an external service to manage workflows (i.e. Nextflow Tower). In fact, I created a script called [kuberun.sh](./kuberun.sh) that does essentially the same thing as `nextflow kuberun`, but avoids some of the issues that `kuberun` has.

### Scheduler warnings

The k8s executor works like other grid executors -- you can queue up as many tasks as you want, and Kubernetes will try to map each task to a node. Unlike other executors, Kubernetes will continuously log warning messages as long as there are un-scheduled pods. Don’t be alarmed by the slurry of warnings, as it is normal. In fact, Kubernetes will even give you a reason for why each node is unavailable, so use that to your advantage where appropriate.

### Diagnosing errors

There are many ways a workflow can fail on Kubernetes. Here are just a few examples:

- Version mismatch between Kubernetes and Nextflow / Java
- “connection timed out” between workflow pod and task pod
- “.command.run not found” when a task is executed, possibly due to PVC latency
- “Unable to fetch pod status”
- “Process terminated for an unknown reason”
- Pod fails with “Error: CPU does not support POPCNT” (my favorite one to date)

Unfortunately, Nextflow’s error handling mechanisms for Kubernetes are not foolproof, and making them foolproof is a continuous process of trial-and-error. Kubernetes can be used on all kinds of different systems, which means that all sorts of different errors can arise. In general, the best way to deal with errors is to catalog them, noting things such as the Nextflow version, the specific error message, when the error happens, how to reproduce (if possible), attempted solutions and whether they worked. This way you can keep track of various errors and determine whether you need to update your code, submit a Nextflow issue, or contact your k8s sysadmin.

One more thing -- when it comes to debugging k8s applications, `kubectl` is your friend. Here are a few commands that I use most often:
```
# view all pods
kubectl get pods [--namespace=<namespace>]

# view all pods, including the node where each pod is running
kubectl get pods -o wide

# look at the yaml config for a pod
kubectl get pod -o yaml <pod-name>

# get information about a pod (similar to previous command)
kubectl describe pod <pod-name>

# get the log output of a pod
kubectl logs [-f] <pod-name>

# get the list of events (useful if a pod was deleted)
kubectl get events
```

### Diagnosing pod failures

When a pod fails, it is usually a result of either (1) an error in the task script, (2) network and/or latency between the workflow pod, task pod, and storage, or (3) some incompatibility between the pod and the underlying hardware (very rare). Aside from the standard procedure of checking the nextflow log and the task directories, you can check the pod metadata by running `kubectl describe <task-pod>`. This metadata will usually have a more detailed description of the error. If a pod fails due to a faulty or incompatible node, you can either (1) use a dynamic retry with backoff and see if the pod lands on a different node or (2) use a node selector or affinity to avoid the node altogether.

Furthermore, some pod failures cause the entire workflow to fail, even when using a retry error strategy. This is because some k8s-related errors are not “wired” into the standard retry logic, and just default to workflow termination. If you encounter such an error, feel free to submit a GitHub issue so that we can evaluate it and hopefully correct it.

### Launching many tasks

In my experience, Nextflow sometimes has issues launching and managing many tasks via k8s. Nextflow will sometimes lose track of a task even though it may have run successfully. Determining the root cause is difficult, as there are a few different errors that can occur. Here are some potential remedies:

- Set the `k8s.computeResourceType` config option to `'Job'`
- Set the `automountServiceAccountToken` pod option to `false`
- Decrease the queue size

The first option (using Jobs instead of Pods) has been particularly effective at increasing the stability of Nextflow / K8s pipelines at scale. For now it is an opt-in feature, but in the future it will likely become the default behavior.

### Requests and limits

Nextflow currently supports requests and limits for accelerators. Similar support for CPUs and memory is in the works.

### Node selector, affinity and labels, tolerations and taints

A __node selector__ allows you to select certain nodes that a pod can use, based on simple expressions such as node name, node type, etc. An __affinity__ does the same thing but allows for more complex expressions such as set logic. You can use these features to control, for example, the CPU or GPU type(s) for a task. You can also control things from the cluster side by adding __labels__ and __taints__ to nodes. A label is just a label and can be anything you want. A taint is a property that will cause a node to repel certain types of pods, for example, a GPU node might have a taint to repel or even reject pods that don’t request any GPUs. You can circumvent a node taint by giving your pod a corresponding __toleration__.

### Requesting new features

Support for k8s features is added on an as-needed basis, so if you need a specific feature, feel free to submit a Feature Request issue and present your use case. Refer to [this document](feature-evaluation.md) to see our own evaluation of which Kubernetes features might be supported in the future.
