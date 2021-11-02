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


## Layout
Pre-configured configuration layouts are as follows

### dev
A single node deployment with the following resources configuration
```
resources:
  requests:
    memory: 1024M
    cpu: 0.25
  limits:
    memory: 1024M
    cpu: 0.25
```
Configuration parameters:
```
- ES_JAVA_OPTS="-Xms512m -Xmx512m"
- cluster.initial_master_nodes=""
- node.store.allow_mmap="false"
- node.roles="master,data"
- discovery.type="single-node"
```
### prod
High-availability 3 node deployment with the following resources configuration
```
resources:
  requests:
    memory: 1024Mi
    cpu: 1
  limits:
    memory: 2048Mi
    cpu: 1
```
Configuration parameters:
```
- ES_JAVA_OPTS="-Xms1024m -Xmx1024m"
- cluster.initial_master_nodes="elasticsearch-0,elasticsearch-1,elasticsearch-2"
- node.store.allow_mmap="false"
- node.roles="master,data"
```

## How to run
To create the highly available version of the deployment, the following command should be run:
```
kubectl kustomize ./manifests/overlays/prod | kubectl apply -f -
```

For testing, a single node deployment can be created by running the following command:
```
kubectl kustomize ./manifests/overlays/prod | kubectl apply -f -
```

## Clean-up
To clean the highly available version, run:
```
kubectl kustomize ./manifests/overlays/prod | kubectl delete -f -
```
For single node:
```
kubectl kustomize ./manifests/overlays/dev | kubectl delete -f -
```

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
For liveness: a tcp probe on port 9300 is configured.
For readiness: an http probe is configured to query port 9200.

### Cluster bootstrap
The cluster is configured with initial master nodes to elasticsearch-0, elasticsearch-1, elasticsearch-2 (when building prod layer).
It's not recommended to create cluster with less than 3 nodes for high availability.

### Resources and pod affinity
Elasticsearch is configured with the following memory limits: 1024m for prod and 512m for dev
While this is not sufficient for a performant cluster, it's enough to deploy the functional elasticsearch service.

The statefulset is configured with pod anti-affinity preference to prevent co-locating elasticsearch nodes on the same hosts. If that is not possible (ie there are less Kubernetes nodes than elasticsearch nodes) then (some or all)  pods will eventually be co-located on the same node.

To guarantee Elasticsearch pods are spread across Kubernetes nodes ensure there are enough Kubernetes nodes in your cluster.

## Deployment verification and troubleshooting
It takes around 5 minutes for cluster pods status to change to Ready.
To verify the cluster status, the following command could be useful:
```
kubectl run debug --rm --restart=Never --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/health
```
The output should display how many nodes and shards are in healthy state.


