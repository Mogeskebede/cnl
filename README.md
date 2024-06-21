# cnl
Subject: Ensuring Single Instance Execution of capi_drs with OpenShift and Consul

I hope this email finds you well. As part of our effort to transition CAPI from Tier 6 to Tier 3, we've identified the need to ensure that the capi_drs microservice runs only in a single instance at any given time. To achieve this, we can leverage OpenShift for orchestration and Consul for service discovery and leader election. Below is a detailed plan on how we can implement this.

Step-by-Step Plan to Ensure Single Instance Execution of capi_drs
Set Up OpenShift Cluster

Ensure we have a functional OpenShift cluster with the necessary nodes and resources.
Install the OpenShift CLI (oc) on our local machines for management tasks.
Deploy Consul on OpenShift

Deploy Consul using Helm:
sh
Copy code
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul --set global.name=consul
Expose the Consul UI and API by creating an OpenShift route:
sh
Copy code
oc expose svc consul-ui
Configure Consul for Service Discovery and Leader Election

Ensure Consul Connect is enabled in the Consul configuration:
json
Copy code
{
  "connect": {
    "enabled": true
  }
}
Create a lock configuration for capi_drs to use Consul’s session and lock mechanism to ensure it runs in only one instance.
Create Deployment Configuration for capi_drs

Define a deployment with a single replica to ensure only one instance is running:
yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: capi-drs
  namespace: capi-drs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: capi-drs
  template:
    metadata:
      labels:
        app: capi-drs
    spec:
      containers:
      - name: capi-drs
        image: your-docker-registry/capi-drs:latest
        ports:
        - containerPort: 8080
        env:
        - name: CONSUL_HTTP_ADDR
          value: "http://consul-server.consul:8500"
Implement Leader Election for High Availability

Create a ConfigMap with a lock script for capi_drs:
yaml
Copy code
apiVersion: v1
kind: ConfigMap
metadata:
  name: capi-drs-lock
  namespace: capi-drs
data:
  consul-lock.sh: |
    #!/bin/sh
    while true; do
      consul lock -name capi-drs-lock /bin/bash -c "start capi-drs"
    done
Mount this script in the capi_drs container and ensure it runs on startup.
Deploy the ConfigMap and Update Deployment

Apply the ConfigMap:
sh
Copy code
oc apply -f capi-drs-lock-configmap.yaml
Update the capi-drs deployment to include the startup script:
yaml
Copy code
containers:
- name: capi-drs
  image: your-docker-registry/capi-drs:latest
  ports:
  - containerPort: 8080
  volumeMounts:
  - name: consul-lock
    mountPath: /consul/config
  command: ["/bin/sh", "-c", "/consul/config/consul-lock.sh && exec <your-main-process>"]
volumes:
- name: consul-lock
  configMap:
    name: capi-drs-lock
Monitor and Maintain the Environment

Set up Prometheus and Grafana for monitoring OpenShift and Consul metrics.
Implement alerts for any issues related to the availability and performance of capi_drs.
By following these steps, we can ensure that the capi_drs service runs only in a single instance, thereby meeting the project's requirements for transitioning to Tier 3. This approach leverages the capabilities of OpenShift and Consul to provide a robust and scalable solution.

Please let me know if you have any questions or need further details.

Best regards,
![image](https://github.com/Mogeskebede/cnl/assets/104947695/31e6a4ab-15f2-476c-a845-2c6191152b31)
![image](https://github.com/Mogeskebede/cnl/assets/104947695/46fe98ca-d37f-42bb-9599-0a21cfbc3328)
![image](https://github.com/Mogeskebede/cnl/assets/104947695/f4d801d4-1291-4756-bb7b-c945e673a2a3)


