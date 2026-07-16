# Fluent Bit

Fluent Bit is a lightweight log processor and forwarder for collecting,
enriching, and shipping log data. This deployment runs Fluent Bit as a
cluster-wide log collector that gathers logs from every node in the
cluster and forwards them to Pitt Digital's central Cribl service.

## What Is Deployed

The manifest in this directory deploys Fluent Bit and its supporting
resources into the `admin-fluent-bit` namespace. The namespace runs 
under the `privileged` Pod Security Standard because Fluent Bit needs
host-level access to read log files off each node.

Fluent Bit runs as a `DaemonSet`. Kubernetes places exactly one Fluent
Bit pod on every node and keeps that guarantee as nodes are added or
removed, so each pod is responsible only for the logs produced on the
node it happens to run on. The pod tolerates all taints, including the
control-plane taints, which means logs are collected from the master
nodes as well as the workers. Each pod mounts several host paths as
read-only (the container log directory `/var/log`, the container
runtime's log directory, and the systemd journal) and reads directly
from the node's filesystem rather than from the Kubernetes API.

To turn raw log lines into structured, searchable records, Fluent Bit
needs to know which pod, namespace, and labels each line belongs to. It
looks this up against the cluster API, so the deployment grants it a
`ServiceAccount` bound to a `ClusterRole` with read-only (`get`, `list`,
`watch`) access to pods and namespaces. This is the only cluster
permission Fluent Bit holds; it can read metadata but cannot modify
anything.

## The Logging Pipeline

Fluent Bit processes logs as a pipeline with three stages: *inputs*
collect log lines from a source, *filters* transform and enrich them,
and an *output* ships the finished records to a destination. The
pipeline for this deployment is defined in the `fluent-bit-config`
`ConfigMap` and is described below in the order records flow through it.

### Inputs

Two sources feed the pipeline. The first tails every container log file
under `/var/log/containers/*.log`, parsing each line with the `cri`
parser to split off the timestamp, stream, and message that the
container runtime prepends. The second reads the node's systemd journal
and keeps only the entries emitted by `kubelet.service` and
`containerd.service`, which captures node-level events that never appear
in a container log. Both inputs track their read position in a small
local database file so that a restarting pod resumes where it left off
instead of re-sending everything.

### Filters

The container records pass through a chain of filters that make them
useful downstream. The `kubernetes` filter is the important one: for
each record it contacts the cluster API and attaches the pod's metadata,
including namespace, pod name, labels, and annotations.

A final filter stamps every record with the identity of the cluster 
it came from, tagging each one with a `tkg_instance` and `tkg_cluster` 
value and nesting them together under a `tkg` key. Because logs from 
every cluster in the environment arrive at the same central endpoint, 
this stamp lets an operator tell them apart.

### Output

All records are shipped to Pitt's central Cribl service. Each
enriched field maps onto a syslog field so the structure survives the
trip: the cluster identifier becomes the syslog hostname, the pod name
becomes the application name, and the Kubernetes metadata, labels,
annotations, and cluster stamp travel as structured data. Delivery is
over UDP, which is fast and imposes no back-pressure on the node, at the
cost of being best-effort - records are not retransmitted if a datagram
is lost in transit.

## Operating

Because Fluent Bit runs as a `DaemonSet`, the first thing to confirm is
that a pod is running and ready on every node:

```bash
kubectl get pods -n admin-fluent-bit -o wide
```

Each pod exposes an HTTP server on port `2020` that reports internal
metrics; records processed, bytes shipped, and per-output error and
retry counts. These metrics are annotated for Prometheus scraping, but
you can also read them directly from a pod to spot-check that logs are
flowing and that the syslog output is not accumulating errors:

```bash
kubectl exec -it -n admin-fluent-bit <pod-name> -- \
  curl -s http://localhost:2020/api/v1/metrics/prometheus
```

If logs are not arriving in Cribl, the pod's own logs will usually show
the reason — a metadata lookup failing against the API, or the output
plugin reporting connection errors:

```bash
kubectl logs -n admin-fluent-bit <pod-name>
```

### Important Notes

- The cluster identity (`crcd-dev-01`) is hard-coded in the record
  filter. When cloning this deployment to another cluster, update both
  the `tkg_instance` and `tkg_cluster` records so its logs are labelled
  correctly at the destination.
- Delivery to Cribl is over UDP and is therefore best-effort. Fluent Bit
  cannot guarantee, retry, or confirm delivery of records once they
  leave the node, so the absence of a log line in Cribl does not by
  itself indicate a problem with Fluent Bit.
- The `admin-fluent-bit` namespace is deliberately `privileged`. Fluent
  Bit reads node log files through host-path mounts, which the more
  restrictive Pod Security Standards would forbid.
- A pod collects only the logs of the node it runs on. A node with no
  Fluent Bit pod (for example, one whose taints are not covered by the
  configured tolerations) produces no logs at the destination.
