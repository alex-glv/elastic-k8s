# Design
The deployment is implemented using `Statefulset` kubernetes resource type.
StatefulSet allows stable network identity supported by headless `Service`

This allows binding stable volumes to the pods and ensuring they will be bound to the same volume in case a pod dies or evicted and starts again.

While this allows for a robust deployments, there are trade offs
For example, Rolling live upgrade will be allowed from highest-ordinal only. 

## PodManagementPolicy
A podManagementpolicy is set to Parallel so as to allow quicker creation of the cluster by letting StatefulSet create pods in parallell, rather than ordered.

## Health check
For liveness: a tcp probe on port 9300.
For readiness: an http probe on port 9200.

## Cluster bootstrap
The cluster is configured with initial master nodes to `elasticsearch-0`, `elasticsearch-1`, `elasticsearch-2`
It's not recommended to create cluster with less than 3 nodes for high availability purposes.

## Pod affinity
The statefulset is configured with pod anti-affinity preference to prevent co-locating elasticsearch nodes on the same hosts. If that is not possible (ie there are less Kubernetes nodes than elasticsearch nodes) then (some or all)  pods will eventually be co-located on the same node.

To guarantee Elasticsearch pods are spread across Kubernetes nodes ensure there are enough Kubernetes nodes in your cluster.

## Live upgrades
Live upgrades can be performed on a live cluster.
An example command can be used for rollingupgrade:
```
kubectl patch statefulset -n elasticsearch elasticsearch --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"docker.elastic.co/elasticsearch/elasticsearch:7.15.1"}]'
```

However, this does not satisfy high-availability requirement as the cluster state can turn yellow if a node that's being updated holds active shards.

## Live upgrade with high-availability
A more appropriate measure would be to patch statefulset spec `updateStrategy ` partition to 3, this will allow updating the statefulset (eg image, resources) and have all pods still running.

Then, we can exclude shards from one pod, and delete, wait for it have status `Running` and proceed with the next pod.

Here's how it would look in practice:

Change `updatestrategy` partition to 3
```
kubectl patch statefulset -n elasticsearch elasticsearch -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":3}}}}'
```
This will prevent deleting nodes with ordinal lower than 3 (so, all nodes of our 3 node cluster will remain)

Patch the statefulset (in this case, we'll update the image):
with new resource
```
kubectl patch statefulset -n elasticsearch elasticsearch --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"docker.elastic.co/elasticsearch/elasticsearch:7.15.1"}]'
```

Let's exclude a node we are about to upgrade:
```
kubectl run debug --rm --quiet --restart=Never -n elasticsearch --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/settings -XPUT --header "Content-Type: application/json" --data '{
  "transient" : {
    "cluster.routing.allocation.exclude._name" : "elasticsearch-2"
  }
}'
```
And change the partition to 2

```
kubectl patch statefulset -n elasticsearch elasticsearch -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
```

This will upgrade the highest ordinal node - elasticsearch-2

Continuing this process until partition 0 will update all nodes while manitaining cluster health.

The last step will remove node exclusion:
```
kubectl run debug-2 --rm --quiet --restart=Never -n elasticsearch --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/settings -XPUT --header "Content-Type: application/json" --data '{
  "transient" : {
    "cluster.routing.allocation.exclude._name" : ""
  }
}'
```

It's possible to add extra replica before upgrade procedure to have 3 nodes available at all times during upgrade.

## Back-up and recovery
Since we are using persistantvolumes and pvc's, the backup could be achieved by backing up the volumes and creating daily/hourly snapshots of our data volumes.
