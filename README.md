# Nextflow on Kubernetes: Best Practices

## Introduction

This document is a collaborative space for users and developers to contribute accumulated knowledge from running Nextflow on Kubernetes. Anyone is welcome to add content as they see fit and I (@bentsherman) will try to organize everything as it develops.

## When to Use Kubernetes

Kubernetes is a container orchestration system that can run atop on-prem hardware or in the cloud. In some cases, Kubernetes may only add an unnecessary layer of abstraction. For example, to run Nextflow on AWS, you can either provision an EKS cluster and run Nextflow with Kubernetes, or you can use AWS Batch directly. Both approaches support containers, so the EKS cluster may only introduce more overhead without much benefit. On the other hand, if you have an on-prem system with no cloud management layer (e.g. OpenStack), then you might consider using Kubernetes instead of a traditional HPC scheduler such as SLURM.

In the case of cloud-based Kubernetes, here are some other questions to consider:

- Do you want to use any Kubernetes-specific features (e.g. auto-scaling library or a special type of storage) ?
- Does your workflow execute many short-running tasks or a few long-running tasks? The cloud executors are not as efficient in the former case.
- Do you want VMs to be provisioned on the fly (cloud batch) or provision a fixed set of nodes that run pods in parallel (Kubernetes)?

## Summary of Nextflow Documentation

Refer to the [Nextflow documentation](https://www.nextflow.io/docs/latest/kubernetes.html) for information on how to use Nextflow with Kubernetes. The summary of the documentation is as follows:

- Pipeline should be available via Git registry (e.g. GitHub) and containerized via Docker registry (e.g. DockerHub)
- Kubernetes cluster should have a ReadWriteMany PVC and a service account with “view” and “edit” roles
- To run a pipeline: (1) create a submitter pod, (2) fetch Nextflow pipeline, (3) launch pipeline with k8s executor and k8s PVC settings
- Nextflow kuberun automates the above process, but is an experimental feature
- For non-trivial deployments, submitter pod should be managed by an external service (e.g. Nextflow Tower) instead of kuberun
- The pod directive allows a subset of config objects to be passed to container (environment variables, volume claims, node selector, etc)

## Tips, Tricks, and Caveats

### Preparing your pipeline for Kubernetes

You will most likely not need to add any k8s-specific settings to your pipeline to make it work with k8s. Most of the work is just publishing and containerizing your pipeline, which is a good practice on its own. Preparing for k8s is more about configuring your k8s cluster to work with Nextflow, that is, providing a PVC and a service account with appropriate privileges.

### Accessing your PVC

Nextflow and all of its tasks store everything on your PVC. You can use `nextflow kuberun login -v <pvc>` to browse your PVC for debugging purposes. This command is simply a shortcut for creating a pod with the PVC attached. Alternatively, you can just exec into any running workflow pod, for example `kubectl exec -it <run-name> – bash`.

### Using kuberun

Launching pipelines with kuberun is convenient, but it is experimental and should only be used for testing. For example, some workflow variables such as “$baseDir” will not work with kuberun when used in a config file, even when the config file is in the pipeline repository. You also cannot run local pipeline scripts with kuberun, the pipeline must be available from a Git registry. You can provide a local config file, in which case kuberun will provide the config file to the submitter pod as a ConfigMap.

In cases where kuberun doesn’t work for you, you can always get around it by either (1) creating your own submitter pod (like a head node on a HPC system) or (2) using an external service to manage workflows (i.e. Nextflow Tower). See [this example](https://github.com/SystemsGenetics/kube-runner/blob/master/kube-run.sh) for how to create your own submitter node for launching workflows.

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

### Diagnosing pod failures

When a pod fails, it is usually a result of either (1) an error in the task script, (2) network and/or latency between the workflow pod, task pod, and storage, or (3) some incompatibility between the pod and the underlying hardware (very rare). Aside from the standard procedure of checking the nextflow log and the task directories, you can check the pod metadata by running `kubectl describe <task-pod>`. This metadata will usually have a more detailed description of the error. If a pod fails due to a faulty or incompatible node, you can either (1) use a dynamic retry with backoff and see if the pod lands on a different node or (2) use a node selector or affinity to avoid the node altogether.

Furthermore, some pod failures cause the entire workflow to fail, even when using a retry error strategy. This is because some k8s-related errors are not “wired” into the standard retry logic, and just default to workflow termination. If you encounter such an error, feel free to submit a GitHub issue so that we can evaluate it and hopefully correct it.

### Launching many tasks

In my experience, Nextflow sometimes has issues launching and managing many tasks via k8s. Nextflow will sometimes lose track of a task even though it may have run successfully. Determining the root cause is difficult, as there are a few different errors that can occur. Here are some potential remedies:

- Set the `automountServiceAccountToken` pod option to `false`
- Decrease the queue size

### Requests and limits

Nextflow currently supports requests and limits for accelerators. Similar support for CPUs and memory is in the works.

### Node selector, affinity, labels, and taints

A node selector allows you to select certain nodes that a pod can use, based on simple expressions such as node name, node type, etc. An affinity does the same thing but allows for more complex expressions such as set logic. You can use these features to control, for example, the CPU or GPU type(s) for a task. You can also control things from the cluster side by adding labels and taints to nodes. A label is just a label and can be anything you want. A taint is a property that will cause a node to repel certain types of pods, for example, a GPU node might have a taint to repel or even reject pods that don’t request any GPUs.

### Requesting new features

Support for k8s features is added on an as-needed basis, so if you need a specific feature, feel free to submit a Feature Request issue and present your use case. Refer to [this document](feature-evaluation.md) to see our own evaluation of which Kubernetes features might be supported in the future.

## Additional Resources

Nextflow/k8s development:
- [Open PRs](https://github.com/nextflow-io/nextflow/pulls?q=is%3Aopen+is%3Apr+label%3Aplatform%2Fk8s)
- [Open Issues](https://github.com/nextflow-io/nextflow/issues?q=is%3Aopen+is%3Aissue+label%3Aplatform%2Fk8s)

Related Nextflow/k8s projects (by @bentsherman and colleagues)
- https://github.com/SystemsGenetics/kube-runner
- https://github.com/SciDAS/nextflow-api
- https://github.com/cbmckni/kinc-k8s-demo
- https://github.com/cbmckni/gpn-workshop
