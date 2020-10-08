---
title: Using the Kubernetes Operator
weight: 3
---
<!-- TODO: Most of this is much the same as the quickstart, maybe combine, or make this the next start? -->
[The Kubernetes operator](https://operator.m3db.io/) helps you deploy M3 to a cluster. It is a more streamlined setup that uses [custom resource definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to automatically handle operations such as managing cluster placements.

{{% notice warning %}}
The Operator is **alpha software**, and as such its APIs and behavior are subject to breaking changes. While we
aim to produce thoroughly tested reliable software there may be undiscovered bugs.
{{% /notice %}}

The M3 operator aims to automate everyday tasks around managing M3. Specifically, it aims to automate:

-   Creating M3 clusters
-   Destroying M3 clusters
-   Expanding clusters (adding instances)
-   Shrinking clusters (removing instances)
-   Replacing failed instances

It does not try to automate every single edge case a user may ever run into. For example, it does not aim to
automate disaster recovery if an entire cluster is taken down. Such use cases may still require human intervention, but
the operator aims to not conflict with such operations a human may have to take on a cluster.

Generally speaking, the operator's philosophy is if **it would be unclear to a human what action to take, we try not to guess.**

For more background on the operator, see our [KubeCon keynote](https://kccna18.sched.com/event/Gsxn/keynote-smooth-operator-large-scale-automated-storage-with-kubernetes-celina-ward-software-engineer-matt-schallert-site-reliability-engineer-uber) on its origins and usage at Uber.

## Prerequisites

### Kubernetes Versions

The M3 operator targets Kubernetes 1.11 and 1.12, we typically target the two most recent minor Kubernetes versions supported by GKE.

### Multi-Zone Kubernetes Cluster

The M3 operator is intended for use with Kubernetes clusters that span at least 3 zones within a region to create
highly available clusters and maintain quorum in the event of region failures. Instructions for creating regional
clusters on GKE can be found [here](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters).

### An etcd cluster

M3DB stores its cluster placements and all other runtime metadata in [etcd](https://etcd.io).

For testing or non-production use cases, we provide manifests for running etcd on Kubernetes in our [example manifests](https://github.com/m3db/m3db-operator/tree/master/example/etcd). There is one for running ephemeral etcd containers and one for running etcd using basic persistent
volumes. If you use the _etcd-pd.yaml_ manifest, we recommend you change it to use a `StorageClass` equivalent to your
cloud provider's fastest remote disk (such as `pd-ssd` on GCP).

For production use cases, we recommend running etcd the following ways (in order of preference):

1.  External to your Kubernetes cluster to avoid circular dependencies.
2.  Using the [etcd operator](https://github.com/coreos/etcd-operator).

## Installation

### Helm

1.  Add the `m3db-operator` repo:

    ```shell
    helm repo add m3db https://m3-helm-charts.storage.googleapis.com/stable
    ```

2.  Install the `m3db-operator` chart:

    ```shell
    helm install m3db/m3db-operator --namespace m3db-operator
    ```

{{% notice note %}}
If you're uninstalling an instance of the operator previously installed with Helm, you may need to delete some resources such as the ClusterRole, ClusterRoleBinding, and ServiceAccount manually.
{{% /notice %}}

### Manually

Install the bundled operator manifests in the current namespace:

```shell
kubectl apply -f https://raw.githubusercontent.com/m3db/m3db-operator/master/bundle.yaml
```

## Create an M3 Cluster
### "Basic" Cluster

The following creates an M3DB cluster spread across 3 zones, with each M3DB instance able to store up to 350gb of
data using the Kubernetes cluster's default storage class. 

It creates three isolated groups for nodes, each with one node instance. In a production environment you can use a variety of different options to define how nodes are spread across groups based on factors such as resource capacity, or location.

configuration/node_affinity

For examples of different cluster topologies, such as zonal
clusters, see the docs on [node affinity][node-affinity].