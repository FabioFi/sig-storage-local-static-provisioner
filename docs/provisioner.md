# local-volume-provisioner

local-volume-provisioner is an out-of-tree static provisioner for the
Kubernetes [local volume](https://kubernetes.io/docs/concepts/storage/volumes/#local), which is GA feature since 1.14.

It runs on each node in the cluster and monitors specified directories to look for
new local file-based volumes.  The volumes can be a mount point or a directory in
a shared filesystem.  It then statically creates a Local PersistentVolume for each
local volume.  It also monitors when the PersistentVolumes have been released, and
will clean up the volume, and recreate the PV.

## Table of Contents

- [Changelog](#changelog)
- [Design](#design)
- [Prometheus Metrics](#prometheus-metrics)

## Changelog

See [CHANGELOG.md](../CHANGELOG.md).

## Design

There is one provisioner instance on each node in the cluster.  Each instance is
responsible for monitoring and managing the local volumes on its node.

The basic components of the provisioner are as follows:

- Discovery: The discovery routine periodically reads the configured discovery
  directories and looks for new mount points that don't have a PV, and creates
  a PV for it.

- Deleter: The deleter routine is invoked by the Informer when a PV phase changes.
  If the phase is Released, then it cleans up the volume and deletes the PV API
  object.

- Cache: A central cache stores all the Local PersistentVolumes that the provisioner
  has created.  It is populated by a PV informer that filters out the PVs that
  belong to this node and have been created by this provisioner.  It is used by
  the Discovery and Deleter routines to get the existing PVs.

- Controller: The controller runs a sync loop that coordinates the other components.
  The discovery and deleter run serially to simplify synchronization with the cache
  and create/delete operations.

## Monitoring

A dedicated HTTP server (default listening on 0.0.0.0:8080) exposes metrics and
readiness state. 

### Metrics

The metrics are exported through the Prometheus golang client on the path `/metrics`.

| Metric name                                                   | Metric type | Labels                                                                                                                                                                             |
| ----------                                                    | ----------- | -----------                                                                                                                                                                        |
| local_volume_provisioner_persistentvolume_discovery_total     | Counter     | `mode`=&lt;persistentvolume-mode&gt;                                                                                                                                               |
| local_volume_provisioner_persistentvolume_discovery_duration_seconds   | Histogram   | `mode`=&lt;persistentvolume-mode&gt;                                                                                                                                               |
| local_volume_provisioner_persistentvolume_delete_total        | Counter     | `mode`=&lt;persistentvolume-mode&gt; <br> `type`=&lt;process&#124;job&gt;                                                                                                          |
| local_volume_provisioner_persistentvolume_delete_failed_total | Counter     | `mode`=&lt;persistentvolume-mode&gt; <br> `type`=&lt;process&#124;job&gt;                                                                                                          |
| local_volume_provisioner_persistentvolume_delete_duration_seconds      | Histogram   | `mode`=&lt;persistentvolume-mode&gt; <br> `type`=&lt;process&#124;job&gt; <br> `capacity`=&lt;volume-capacity-breakdown-by-500G&gt; <br> `cleanup_command`=&lt;cleanup-command&gt; |
| local_volume_provisioner_apiserver_requests_total             | Counter     | `method`=&lt;request-method&gt;                                                                                                                                                    |
| local_volume_provisioner_apiserver_requests_failed_total      | Counter     | `method`=&lt;request-method&gt;                                                                                                                                                    |
| local_volume_provisioner_apiserver_requests_duration_seconds           | Histogram   | `method`=&lt;request-method&gt;                                                                                                                                                    |
| local_volume_provisioner_proctable_running                    | Gauge       |                                                                                                                                                                                    |
| local_volume_provisioner_proctable_failed                     | Gauge       |                                                                                                                                                                                    |
| local_volume_provisioner_proctable_succeeded                  | Gauge       |                                                                                                                                                                                    |

### Readiness

The readiness state is exposed on the path `/ready`.

The state become ready when discovered local volumes are successfully created.

Note that if there is no disk to create, the state will be marked as ready.
