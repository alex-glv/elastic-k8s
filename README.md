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
  1. get/create/delete deployment 
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

All resources will be created in `elasticsearch` namespace.

## Deployment verification and troubleshooting
It takes around 2 minutes for cluster pods to become Ready.

You can also run kubectl command to track pods status:
```
kubectl get pods --selector=service=elasticsearch  -n elasticsearch -w
```
The output should be similar to this:
```
% kubectl get pods --selector=service=elasticsearch  -n elasticsearch -w
NAME              READY   STATUS    RESTARTS   AGE
elasticsearch-0   0/1     Running   0          27s
elasticsearch-1   0/1     Running   0          27s
elasticsearch-2   0/1     Running   0          27s
```

After all pods turned to Ready, you can veriy cluster status with elasticsearch api.

```
kubectl run debug --rm --quiet --restart=Never -n elasticsearch --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/health | jq
```
The output should display how many nodes and shards are in healthy state.

```
 % kubectl run debug --rm --quiet --restart=Never -n elasticsearch --image curlimages/curl -ti -- curl elasticsearch:9200/_cluster/health | jq
{
  "cluster_name": "elasticsearch",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 6,
  "active_shards": 12,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 2,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 623,
  "active_shards_percent_as_number": 100
}
```

### Kibana
Kibana deployment is running alongside elasticsearch and configured to use the elasticsearch service endpoint.
To access kibana, run
```
kubectl port-forward svc/kibana 5601:5601
```
Kibana should be accessible from `localhost:5601`

## Clean-up
To clean the highly available version, run:
```
kubectl kustomize ./manifests/overlays/prod | kubectl delete -f -
```

## Production use limitations
The following caveats should be considered with regards to Elasticsearch configuration

###  vm.max_map_count is not set to what Elasticsearch recommends for optimal performance
As per [k8s-virtual-memory.html](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-virtual-memory.html) suggestion, `vm.max_map_count` default value on Linux should be increased, however this requires to run priveleged containers, which is outside of the scope of this deployment requirements. Therefore this deployment disables `node.store.allow_mmap` setting.

### Health check
It is recommended to have lenient health checks on 9200 port api as during high load this api might be slow to respond. 

## Configuration changes
To customize the deployment to tailor to your needs, consider creating another overlay.

Copy the `manifests/overlays/prod` to another folder, eg `manifests/overlays/custom`

Now, open `manifests/overlays/custom/kustomization.yaml`, in `configMapGenerator` section you can customize roles and jvm options.

To change the elasticsearch or kibana image, in the `kustomization.yaml ` define a new tag and/or name to override the base image (if you store images in a private registry) 

To change the resources limits, edit the `patch.statefulset.resources.yaml` file.
