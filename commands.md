##### Deploy autoscaling component
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

##### Deploy nginx pods with service
```
kubectl apply -f nginx.yaml
```

##### Updated autoscaling configuration
```
cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
- —node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/NAME-OF-CLUSTER
- —balance-similar-node-groups
- —skip-nodes-with-system-pods=false
image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.XX.X
```