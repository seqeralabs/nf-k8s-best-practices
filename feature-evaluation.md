# Evaluation of Kubernetes Features

This section provides a high-level view of the space of Kubernetes features that may be supported by Nextflow, and our rationale for including or excluding specific features. We hope this guide will help you navigate the sea of features and settings provided by Kubernetes and learn our best practices for using them with Nextflow.

## JobSpec ([source](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#jobspec-v1-batch))

Since we are considering changing Nextflow to use Jobs instead of Pods to submit tasks, here we evaluate how each JobSpec field should be used by Nextflow if this change is made.

| Name                      | Notes                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------- |
| `activeDeadlineSeconds`   | not supported; could be used to facilitate a walltime limit via `time` directive                        |
| `backoffLimit`            | not supported; could be used by `errorStrategy` directive but might be messy                            |
| `completions`             | should always be 1 or nil                                                                               |
| `manualSelector`          | not supported                                                                                           |
| `parallelism`             | should always be 1                                                                                      |
| `selector`                | not supported                                                                                           |
| `template`                | should be used by Nextflow to define pod spec                                                           |
| `ttlSecondsAfterFinished` | not supported; could be used to automatically delete finished jobs but is still an experimental feature |

## PodSpec ([source](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#podspec-v1-core))

The following section shows the state of Nextflow support for each PodSpec field. Features that are not supported and do not have any additional notes do not have a strong enough use case to be supported by Nextflow. If you have a use case that is not documented here, feel free to submit an issue here or (preferably) at [nextflow-io/nextflow](https://github.com/nextflow-io/nextflow/issues).

| Name                            | Notes                                                                            |
| ------------------------------- | -------------------------------------------------------------------------------- |
| `activeDeadlineSeconds`         | not supported; could be used to facilitate a walltime limit via `time` directive |
| `affinity`                      | already supported                                                                |
| `automountServiceAccountToken`  | already supported                                                                |
| `containers`                    | already supported via `container` directive                                      |
| `dnsConfig`                     | not supported                                                                    |
| `dnsPolicy`                     | not supported                                                                    |
| `enableServiceLinks`            | not supported                                                                    |
| `ephemeralContainers`           | not supported                                                                    |
| `hostAliases`                   | not supported                                                                    |
| `hostIPC`                       | not supported                                                                    |
| `hostNetwork`                   | not supported                                                                    |
| `hostPID`                       | not supported                                                                    |
| `hostname`                      | not supported                                                                    |
| `imagePullSecrets`              | already supported                                                                |
| `initContainers`                | not supported                                                                    |
| `nodeName`                      | not supported; user should use `nodeSelector` pod option instead                 |
| `nodeSelector`                  | already supported (original object syntax and custom string syntax)              |
| `overhead`                      | not supported                                                                    |
| `preemptionPolicy`              | not supported; might be useful; easy to implement                                |
| `priority`                      | not supported; use `priorityClassName` instead                                   |
| `priorityClassName`             | already supported                                                                |
| `readinessGates`                | not supported; might be useful; easy to implement                                |
| `restartPolicy`                 | not supported; user should use `errorStrategy` directive instead                 |
| `runtimeClassName`              | not supported; might be useful; easy to implement                                |
| `schedulerName`                 | not supported; might be useful; easy to implement                                |
| `securityContext`               | already supported                                                                |
| `serviceAccountName`            | already supported via `k8s.serviceAccount`                                       |
| `setHostnameAsFQDN`             | not supported                                                                    |
| `shareProcessNamespace`         | not supported                                                                    |
| `subdomain`                     | not supported                                                                    |
| `terminationGracePeriodSeconds` | not supported; might be useful; easy to implement                                |
| `tolerations`                   | not supported; probably should be supported alongside `affinity`, `nodeSelector` |
| `topologySpreadConstraints`     | not supported; might be useful; easy to implement                                |
| `volumes`                       | already supported (see below)                                                    |

## Container ([source](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#container-v1-core))

| Name                       | Notes                                           |
| -------------------------- | ----------------------------------------------- |
| `args`                     | already used by Nextflow                        |
| `command`                  | already used by Nextflow                        |
| `env`                      | already supported                               |
| `envFrom`                  | not supported; user should use `env` pod option |
| `image`                    | already used by Nextflow                        |
| `imagePullPolicy`          | already supported                               |
| `lifecycle`                | not supported                                   |
| `livenessProbe`            | not supported                                   |
| `name`                     | already used by Nextflow                        |
| `ports`                    | not supported                                   |
| `readinessProbe`           | not supported                                   |
| `resources`                | already used by Nextflow                        |
| `securityContext`          | already supported                               |
| `startupProbe`             | not supported                                   |
| `stdin`                    | not supported                                   |
| `stdinOnce`                | not supported                                   |
| `terminationMessagePath`   | not supported; might be useful internally       |
| `terminationMessagePolicy` | not supported; might be useful internally       |
| `tty`                      | not supported                                   |
| `volumeDevices`            | not supported                                   |
| `volumeMount`              | already supported                               |
| `workingDir`               | not supported                                   |

## Volume ([source](https://kubernetes.io/docs/concepts/storage/volumes/))

In general, Nextflow does not support specific storage backends (see [nextflow-io/nextflow #998](https://github.com/nextflow-io/nextflow/issues/998)). It is recommended that you provision any storage through a PersistentVolumeClaim and use the `volumeClaim` pod option.

| Name                    | Notes                                                                 |
| ----------------------- | --------------------------------------------------------------------- |
| `awsElasticBlockStore`  | not supported                                                         |
| `azureDisk`             | not supported                                                         |
| `azureFile`             | not supported                                                         |
| `cephfs`                | not supported                                                         |
| `cinder`                | not supported                                                         |
| `configMap`             | already supported                                                     |
| `csi`                   | not supported; might be useful for local disk space                   |
| `downwardAPI`           | not supported; might be useful; similar to `env/fieldPath` pod option |
| `emptyDir`              | not supported; might be useful for local disk space                   |
| `ephemeral`             | not supported; might be useful for local disk space                   |
| `fc (fibre channel)`    | not supported                                                         |
| `flexVolume`            | not supported                                                         |
| `gcePersistentDisk`     | not supported                                                         |
| `glusterfs`             | not supported                                                         |
| `hostPath`              | not supported                                                         |
| `iscsi`                 | not supported                                                         |
| `local`                 | not supported                                                         |
| `nfs`                   | not supported                                                         |
| `persistentVolumeClaim` | already supported                                                     |
| `photonPersistentDisk`  | not supported                                                         |
| `portworxVolume`        | not supported                                                         |
| `projected`             | not supported                                                         |
| `rbd`                   | not supported                                                         |
| `secret`                | already supported                                                     |
| `vsphereVolume`         | not supported                                                         |

## Recommendations

Based on the above evaluation, I recommend the following changes:

- Rename `pod` directive to `podOptions` to emphasize the analogy to `clusterOptions`
- Use `activeDeadlineSeconds` to facilitate walltime limits for k8s tasks via `time` directive
- Add support for the following pod options: `tolerations`, `downwardAPI`, `emptyDir`
- Deprecate `runAsUser` pod option in favor of `securityContext`
