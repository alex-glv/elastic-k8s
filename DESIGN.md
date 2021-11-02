# Design
The deployment is implemented using `Statefulset` kubernetes resource type.
StatefulSet allows stable network identity supported by headless `Service`

This allows binding stable volumes to the pods and ensuring they will be bound to the same volume in case a pod dies or evicted and starts again.

While this allows for a robust deployments, there are trade offs

## Live upgrade
Live upgrades can be performed on a live cluster.
An example command can be used for rollingupgrade:
```
kubectl patch statefulset -n elasticsearch elasticsearch --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"docker.elastic.co/elasticsearch/elasticsearch:7.15.1"}]'
```

However, this does not satisfy high-availability requirement as the cluster state can turn yellow if a node that's being updated holds active shards.

## Live upgrade with high-availability
A more appropriate measure would be to patch statefulset spec `updatestrategy ` partition to 3, this will allow updating the statefulset (eg image, resources) and have all pods still running.

Then, we can move shards from one pod, and delete, wait for it have status `Running` and proceed with the next pod.

Now, let's exclude a node we are about to upgrade:
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

It's possible to add extra replica before upgrade procedure to have 3 nodes available at all times during upgrade.
