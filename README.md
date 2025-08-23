# Dragonfly Database

# DragonflyDB Install and Configuration

Dragonfly is a modern in-memory datastore, fully compatible with Redis and Memcached APIs. Dragonfly implements novel algorithms and data structures on top of a multi-threaded, shared-nothing architecture. As a result, Dragonfly reaches 25X performance compared to Redis and supports millions of QPS on a single instance.

# **Installation (Operator Method)**

Make sure your Kubernetes cluster is up and running. To install Dragonfly Operator, run the following command:

```bash
kubectl apply -f https://raw.githubusercontent.com/dragonflydb/dragonfly-operator/main/manifests/dragonfly-operator.yaml
```

By default, the operator will be installed in the `dragonfly-operator-system` namespace.

## **Create a Dragonfly instance with replicas**

To set up a sample Dragonfly topology with a primary (master) and optional replicas (slaves), create a YAML file:

```yaml
# dragonfly-sample.yaml
apiVersion: dragonflydb.io/v1alpha1
kind: Dragonfly
metadata:
  labels:
    app.kubernetes.io/name: dragonfly
    app.kubernetes.io/instance: dragonfly-sample
    app.kubernetes.io/part-of: dragonfly-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: dragonfly-operator
  name: dragonfly-sample
spec:
  replicas: 2
  resources:
    requests:
      cpu: 500m
      memory: 500Mi
    limits:
      cpu: 600m
      memory: 750Mi
```

And then run the following command:

```bash
kubectl apply -f dragonfly-sample.yaml
```

To check the status of the instance, run:

```bash
kubectl describe dragonflies.dragonflydb.io dragonfly-sample
```

![image.png](attachment:c73b1f1a-9e53-42f0-ae26-40e6b0ce910f:image.png)

Connect to the master instance of the service at:

`<dragonfly-name>.<namespace>.svc.cluster.local`.

As pods are added or removed, the service automatically updates to point to the new master.
The Operator creates a **StatefulSet** for your Dragonfly instance.

## Create NodePort service (Optional):

```yaml
# dragonfly-master-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: dragonfly-master-nodeport
  namespace: default
spec:
  selector:
    role: master        # selects the current master pod
  type: NodePort
  ports:
    - port: 6379         # pod port
      targetPort: 6379
      nodePort: 30079    # external NodePort
```

Apply it:

```bash
kubectl apply -f dragonfly-master-nodeport.yaml
```

After that we can connect to external host or application like Another Redis Desktop Manager.

<img width="1133" height="661" alt="image" src="https://github.com/user-attachments/assets/657124c2-2e61-4994-842e-b60646bc8e83" />
