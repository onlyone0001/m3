---
title: Quickstart with Kubernetes
weight: 3
---

{{% notice tip %}}
We recommend you use [our Kubernetes operator](https://operator.m3db.io/) to deploy M3 to a cluster. It is a more streamlined setup that uses [custom resource definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to automatically handle operations such as managing cluster topology.
{{% /notice %}}

This guide shows you how to create an M3DB cluster of 3 nodes, designed to run locally on the same machine. It is designed to show you how M3DB and Kubernetes can work together, but not as a production example.

## Prerequisites

-   A running Kubernetes cluster.
    -   For local testing, you can use [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/), [Docker desktop](https://www.docker.com/products/docker-desktop), or [we have a script](https://raw.githubusercontent.com/m3db/m3db-operator/master/scripts/kind-create-cluster.sh) you can use to start a 3 node cluster with [Kind](https://kind.sigs.k8s.io/docs/user/quick-start/).
-   To use the M3DB operator chart, you need [Helm](https://helm.sh/)

## Create an etcd cluster

M3DB stores its cluster topology and runtime metadata in [etcd](https://etcd.io) and needs a running cluster to communicate with.

We have example services and stateful set you can use, but feel free to use your own configuration and change any later instructions accordingly.

```shell
kubectl apply -f https://github.com/m3db/m3db-operator/blob/v0.9.0/example/etcd/etcd-minikube.yaml
```

If the etcd cluster is running on your local machine, update your _/etc/hosts_ file to match the domains specified in the `etcd` `--initial-cluster` argument. For example to match the `StatefulSet` declaration in the _etcd-minikube.yaml_ above, that is:

```text
$(minikube ip) etcd-0.etcd
$(minikube ip) etcd-1.etcd
$(minikube ip) etcd-2.etcd
```

Verify that the cluster is running with something like the Kubernetes dashboard, or the command below:

```shell
kubectl exec etcd-0 -- env ETCDCTL_API=3 etcdctl endpoint health
```

## Install the Operator with Helm

1.  Add the m3db operator repo:

```shell
helm repo add m3db https://m3-helm-charts.storage.googleapis.com/stable
```

2.  Install the m3db operator chart:

```shell
helm install m3db-operator m3db/m3db-operator
```

{{% notice note %}}
If uninstalling an instance of the operator installed with Helm, you may need to delete some resources such as the ClusterRole, ClusterRoleBinding, and ServiceAccount manually.
{{% /notice %}}

## Create an M3DB Cluster

The following creates an M3DB cluster with 3 replicas of data across 256 shards that connects to the 3 available etcd endpoints.

It creates three isolated groups for nodes, each with one node instance. In a production environment you can use a variety of different options to define how nodes are spread across groups based on factors such as resource capacity, or location.

It creates namespaces in the cluster with the `namespaces` parameter. You can use M3DB-provided presets, or define your own. This example creates a namespace with the `10s:2d` preset.

The cluster derives pod identity from the `podIdentityConfig` parameter, which in this case is the UID of the Pod.

[Read more details on all the parameters in the Operator API docs](https://operator.m3db.io/api/).

```shell
kubectl apply -f https://github.com/m3db/m3db-operator/blob/v0.9.0/example/m3db-local.yaml
```

Verify that the cluster is running with something like the Kubernetes dashboard, or the command below:

```shell
kubectl exec simple-cluster-rep2-0 -- curl -sSf localhost:9002/health
```

## Deleting a Cluster

Delete the M3DB cluster using kubectl:

```shell
kubectl delete m3dbcluster simple-cluster
```

By default, the operator uses [finalizers](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#finalizers) to delete the placement and namespaces associated with a cluster before the custom resources. If you do not want this behavior, set `keepEtcdDataOnDelete` to `true` in the cluster configuration.