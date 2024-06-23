oci
If the `Mogi` pod fails, is deleted, or is restarted, OpenShift (or Kubernetes) will automatically attempt to recreate the pod to maintain the desired state as specified in the Deployment configuration. However, to ensure that `Mogi` always runs as a single instance, we need to address the leader election and ensure there's no downtime or multiple instances running concurrently during the pod recreation process.

Here’s what will happen and how we can handle it:

### What Happens When `Mogi` Pod Fails, is Deleted, or Restarted

1. **Pod Failure/Deletion/Restart**:
   - When the `Mogi` pod fails, is deleted, or is restarted, Kubernetes/ OpenShift’s controller manager will detect that the desired state (1 replica) is not met.
   - Kubernetes/OpenShift will then schedule a new pod to replace the failed, deleted, or restarted one.

2. **Consul Lock Release**:
   - The lock held by the failing or deleted `Mogi` pod in Consul will be released. This ensures that no two instances of `Mogi` are running concurrently, as the session associated with the lock will expire.

3. **Pod Recreation**:
   - The recreated pod will start and execute the leader election script to acquire the lock in Consul again. If successful, it will become the active instance of `Mogi`.

### Ensuring Smooth Failover and Leader Election

To ensure that `Mogi` always runs as a single instance and handles failover smoothly, we will use Consul’s session and lock mechanism effectively. Here’s the improved process:

### Step-by-Step Process

1. **Deployment Configuration**:
   - Ensure the deployment is configured to have only one replica of `Mogi`:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: Mogi
       namespace: Mogi
       annotations:
         consul.hashicorp.com/connect-inject: "true"
         consul.hashicorp.com/service-name: "Mogi"
         consul.hashicorp.com/service-port: "8080"
         consul.hashicorp.com/service-check-http: "/health"
         consul.hashicorp.com/service-check-interval: "10s"
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: Mogi
       template:
         metadata:
           labels:
             app: Mogi
           annotations:
             consul.hashicorp.com/connect-inject: "true"
             consul.hashicorp.com/service-name: "Mogi"
             consul.hashicorp.com/service-port: "8080"
             consul.hashicorp.com/service-check-http: "/health"
             consul.hashicorp.com/service-check-interval: "10s"
         spec:
           containers:
           - name: Mogi
             image: your-docker-registry/Mogi:latest
             ports:
             - containerPort: 8080
             volumeMounts:
             - name: consul-lock
               mountPath: /consul/config
             env:
             - name: CONSUL_HTTP_ADDR
               value: "http://consul-server.consul:8500"
             command: ["/bin/sh", "-c", "/consul/config/consul-lock.sh && exec <your-main-process>"]
           volumes:
           - name: consul-lock
             configMap:
               name: Mogi-lock
     ```

2. **ConfigMap for Consul Lock Script**:
   - Create a ConfigMap that includes the lock script to ensure only one instance of `Mogi` is active:
     ```yaml
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: Mogi-lock
       namespace: Mogi
     data:
       consul-lock.sh: |
         #!/bin/sh
         # Acquire the Consul lock
         while true; do
           consul lock -name Mogi-lock /bin/bash -c "start Mogi"
         done
     ```
   - Apply the ConfigMap:
     ```sh
     oc apply -f Mogi-lock-configmap.yaml
     ```

### Enhanced Locking Mechanism

To handle failover and ensure no two instances run concurrently, the lock mechanism will ensure the new pod acquires the lock before starting the main process.

### Handling Failover and Restart

1. **Lock Acquisition**:
   - When the `Mogi` pod starts, it will attempt to acquire the lock from Consul. If the lock is available (because the previous pod has failed or been deleted), the new pod will acquire it and start the `Mogi` service.

2. **Single Instance Guarantee**:
   - If another instance of `Mogi` is already holding the lock, the new instance will wait until the lock becomes available. This ensures only one instance of `Mogi` is running at any time.

### Example Enhanced Lock Script

Here’s an example of the enhanced lock script to ensure `Mogi` runs as a single instance:

```sh
#!/bin/sh
# Enhanced lock script for ensuring single instance of Mogi

# Acquire the Consul lock
while true; do
  consul lock -name Mogi-lock /bin/bash -c "exec <your-main-process>"
done
```

This script will keep trying to acquire the lock, and once acquired, it will start the `Mogi` process. If the process fails, the lock will be released, allowing another instance to acquire it.

### Summary

By using Kubernetes/OpenShift for pod management and Consul for service discovery and leader election, you can ensure that `Mogi` runs as a single instance. In case of a pod failure, deletion, or restart, Kubernetes/OpenShift will recreate the pod, and the new pod will use Consul to ensure only one instance is active. This combination ensures high availability and consistency for the `Mogi` service.
