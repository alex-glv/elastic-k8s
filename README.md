# elastic-k8s
Deploy ElasticSearch cluster on Kubernetes

# Production use limitations
The following caveats should be considered with regards to Elasticsearch configuration:

##  vm.max_map_count is not set to what Elasticsearch recommends for optimal performance
As per [https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-virtual-memory.html](k8s-virtual-memory.html) suggestion, vm.max_map_count default value on Linux should be increased, however this requires to run priveleged containers.
Hence, this deployment disables node.store.allow_mmap setting.

## Lock memory option
<TODO>
