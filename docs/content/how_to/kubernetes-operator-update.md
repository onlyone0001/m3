---
title: Using the Kubernetes Operator to Update a Cluster
weight: 3
---


If you need to make changes to a running cluster, you can use the operator and add an annotation to enable updates.

First update the M3 cluster's [custom resource definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), this is any Kubernetes configuration file beginning with the following:

```yaml
apiVersion: operator.m3db.io/v1alpha1
kind: M3DBCluster
```

The first step in updating a cluster is to update the cluster's `M3DBCluster` CRD with the changes
you want to make. If you manage your cluster via manifests stored in YAML files then this is as
simple as updating the manifest and applying your changes:

```bash
kubectl apply -f example/my-cluster.yaml
```

As a precaution, the operator won't immediately begin updating a cluster after your changes have
been applied. Instead, you'll need to add the following annotation on each `StatefulSet` in the
cluster to indicate to the operator that it is safe to update that `StatefulSet`:

```bash
kubectl annotate statefulset my-cluster-rep0 operator.m3db.io/update=enabled
```

When the operator sees this annotation, it will check if the current state of the `StatefulSet`
differs from its desired state as defined by the `M3DBCluster` CRD. If so, the operator will
update the `StatefulSet` to match its desired state, thereby triggering a rollout of the pods in
the `StatefulSet`. The operator will also remove the `operator.m3db.io/update=enabled` annotation
from the updated `StatefulSet`.

If, on the other hand, the operator finds the update annotation on a `StatefulSet` but it doesn't
need to be updated then the operator will remove the annotation but perform no other actions.
Consequently, once you set the update annotation on a `StatefulSet`, you can watch for the
annotation to be removed from it to know if the operator has seen and checked for an update.

Since M3DB rollouts can take longer periods of time, it's often more convenient to set the
annotation to enable updates on each `StatefulSet` in the cluster at once, and allow the operator
to perform the rollout safely. The operator will update only one `StatefulSet` at a time and then
wait for it to bootstrap and become healthy again before moving onto the next `StatefulSet` in the
cluster so that no two replicas are ever down at the same time.

```bash
kubectl annotate statefulset -l operator.m3db.io/cluster=my-cluster operator.m3db.io/update=enabled
```
