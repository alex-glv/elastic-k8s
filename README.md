# elastic-k8s
Deploy highly-available ElasticSearch cluster on Kubernetes.

## Prerequisites
To deploy a highly-available cluster on Kubernetes the following requirements apply:
- Kubernetes cluster v1.18 or higher

## How to run
To create the cluster in `elasticsearch` namespace, run the following command:
```
kubectl kustomize ./kustomize | kubectl apply -f -
```

The following output indicates succesful resource creation:
```
elastic-k8s % kubectl kustomize ./kustomize | kubectl apply -f -
namespace/elastic created
service/elasticsearch created
statefulset.apps/elasticsearch created
```

## Clean-up
```
kubectl kustomize ./kustomize | kubectl delete -f -
```

The output should be similar to this:
```
elastic-k8s % kubectl kustomize ./kustomize | kubectl delete -f -
namespace "elastic" deleted
service "elasticsearch" deleted
statefulset.apps "elasticsearch" deleted
```

## Production use limitations
The following caveats should be considered with regards to Elasticsearch configuration

###  vm.max_map_count is not set to what Elasticsearch recommends for optimal performance
As per [k8s-virtual-memory.html](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-virtual-memory.html) suggestion, `vm.max_map_count` default value on Linux should be increased, however this requires to run priveleged containers, which is outside of the scope of this deployment requirements. Therefore this deployment disables `node.store.allow_mmap` setting.

### RBAC
A role with the following minimum RBAC is required:
- get/create namespace
- get/create statefulset
- get/create service

### PodManagementPolicy
A podManagementpolicy is set to Parallel so as to allow quicker creation of the cluster by letting StatefulSet create pods in parallell, rather than ordered.


### Health check
For liveness: a tcp probe on port 9300 is configured.
For readiness: an http probe is configured to query port 9200.


### Cluster bootstrap
The cluster is configured with initial master nodes to elasticsearch-0, elasticsearch-1, elasticsearch-2.

It's not recommended to create cluster with less than 3 nodes for high availability.

### Resources and pod affinity
Elasticsearch is configured with 512m memory limits.
While this is not sufficient for a performant cluster, it's enough to deploy the functional elasticsearch service.

The statefulset is configured with pod anti-affinity to prevent co-locating elasticsearch nodes on the same hosts. If that is not possible (ie there are less Kubernetes nodes than elasticsearch nodes) (some or all)  pods will eventually be co-located on the same node.
