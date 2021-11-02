# Elasticsearch on Kubernetes
Deploy highly-available ElasticSearch cluster on Kubernetes.

## Prerequisites
To deploy a highly-available cluster on Kubernetes the following requirements apply:
- Kubernetes cluster v1.18 or higher
- Kubectl v1.18 or higher
- A role with the following minimum RBAC is required:
  1. get/create/delete namespace
  1. get/create/delete statefulset
  1. get/create/delete service
  1. get/create/delete configmap
- (optional) jq


## Layout and structure
kustomize is used to organise the manifests and configuration.
There are two overlays available:
- dev
- prod

The most meaningful difference is `dev` deploys a single-instance cluster for development purposes, while `prod` deploys 3 instance high-availability Elasticsearch cluster. All examples below will use `prod`.

## How to run
To create the highly available version of the deployment, the following command should be run:
```
kubectl kustomize ./manifests/overlays/prod | kubectl apply -f -
```

## Clean-up
To clean the highly available version, run:
```
kubectl kustomize ./manifests/overlays/prod | kubectl delete -f -
```

## Deployment verification and troubleshooting
It takes around 2 minutes for cluster pods status to change to Ready.
To verify the cluster status, the following command can be used:
```
kubectl run debug --rm --quiet --restart=Never --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/health | jq
```
The output should display how many nodes and shards are in healthy state.

## Production use limitations
The following caveats should be considered with regards to Elasticsearch configuration

###  vm.max_map_count is not set to what Elasticsearch recommends for optimal performance
As per [k8s-virtual-memory.html](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-virtual-memory.html) suggestion, `vm.max_map_count` default value on Linux should be increased, however this requires to run priveleged containers, which is outside of the scope of this deployment requirements. Therefore this deployment disables `node.store.allow_mmap` setting.

### Health check
It is recommended to have lenient health checks on 9200 port api as during high load this api might be slow to respond. 

## Configuration details

### PodManagementPolicy
A podManagementpolicy is set to Parallel so as to allow quicker creation of the cluster by letting StatefulSet create pods in parallell, rather than ordered.

### Health check
For liveness: a tcp probe on port 9300.
For readiness: an http probe on port 9200.

### Cluster bootstrap
The cluster is configured with initial master nodes to `elasticsearch-0`, `elasticsearch-1`, `elasticsearch-2`
It's not recommended to create cluster with less than 3 nodes for high availability purposes.

### Pod affinity
The statefulset is configured with pod anti-affinity preference to prevent co-locating elasticsearch nodes on the same hosts. If that is not possible (ie there are less Kubernetes nodes than elasticsearch nodes) then (some or all)  pods will eventually be co-located on the same node.

To guarantee Elasticsearch pods are spread across Kubernetes nodes ensure there are enough Kubernetes nodes in your cluster.

