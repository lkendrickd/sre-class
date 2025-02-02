# Beginner's Guide to Kubernetes: From Setup to Scaling

Welcome! This hands-on guide will walk you through the fundamentals of Kubernetes. You'll learn how to deploy applications, handle failures, manage storage, and scale your services. Don't worry if you're new to Kubernetes - we'll explain each concept as we go.

## Prerequisites
- Access to a Kubernetes cluster (local or cloud)
- kubectl command-line tool installed
- Basic familiarity with command line operations

## Getting Started

Before we begin, here are some key terms you'll encounter:
- **Pod**: The smallest unit in Kubernetes, usually containing one or more containers
- **Deployment**: Manages multiple copies of your pods
- **Service**: Provides a stable way to access your pods
- **PersistentVolume**: Provides storage that can survive pod restarts

## 1. Initial Setup & Cluster Health Check

### What is Cluster Health?
A Kubernetes cluster is made up of multiple machines (nodes) working together. Before we can run applications, we need to make sure all parts of our cluster are healthy and communicating properly. Think of it like checking all systems in a car before a long journey.

### Why Check Pod Status?
Pods are the smallest deployable units in Kubernetes - they're like small boxes that contain your application containers. A pod that isn't "Running" or has multiple restarts might indicate problems with your application or the cluster itself.

### Goal: Verify the health of the Kubernetes cluster and pods.

First, let's create a basic NGINX deployment:

```bash
# Create an NGINX deployment
kubectl create deployment nginx-demo --image=nginx:latest

# Wait a moment, then verify all pods across namespaces
kubectl get pods --all-namespaces

# Check pods in the default namespace
kubectl get pods -l app=nginx-demo
```

**What these commands do:**
- First command creates a deployment running NGINX web server
- Second command shows ALL pods in ALL namespaces (think of namespaces as separate workspaces or projects)
- Third command filters to show only our NGINX pods
- The `-l` flag means we're filtering by label, which is how Kubernetes organizes related resources

**Expected Outcome**: All pods should be "Running" and ready, with no restarts. You'll see a table showing pod names, their status, and the number of restarts.

## 2. Simulate Pod Failure & Observe Self-Healing

### What is Self-Healing?
One of Kubernetes' most powerful features is its ability to automatically recover from failures. If a pod crashes or a node fails, Kubernetes will automatically try to fix the situation by creating new pods or moving them to healthy nodes.

### What is a ReplicaSet?
A ReplicaSet ensures that a specified number of pod copies (replicas) are running at any given time. If a pod dies, the ReplicaSet automatically creates a new one to maintain the desired count. Think of it like a manager who makes sure there are always enough workers on shift.

### Goal: Test Kubernetes self-healing capabilities using ReplicaSet

```bash
# Scale NGINX to 3 replicas
kubectl scale deployment nginx-demo --replicas=3

# Confirm initial state
kubectl get pods -l app=nginx-demo

# Delete one of the pods (replace with actual pod name from your cluster)
kubectl delete pod nginx-demo-66b6c48dd5-xxxxx

# Watch pod recreation
kubectl get pods -l app=nginx-demo -w

# Optional: Check ReplicaSet details
kubectl describe rs -l app=nginx-demo
```

**What these commands do:**
- First, we scale up to 3 copies of our web server
- Then we check how many pods we currently have
- Next we deliberately delete one pod to simulate a failure
- The `-w` flag lets us watch in real-time as Kubernetes responds
- The last command shows detailed information about the ReplicaSet managing our pods

**Expected Outcome**: When you delete a pod, you'll see the pod count temporarily drop to 2, then quickly return to 3 as the ReplicaSet automatically creates a new pod to replace the deleted one.

## 3. Resource Monitoring

### Why Monitor Resources?
Just like your computer needs enough CPU and memory to run programs, pods need resources to run your applications. Monitoring these resources helps you ensure your applications have what they need and aren't using too much.

### What is the Metrics Server?
The Metrics Server is like a system monitor for your cluster. It collects CPU and memory usage data from all your pods and nodes, making this information available for monitoring and scaling decisions.

### Goal: Monitor pod resources and verify persistent storage

```bash
# Check if Metrics Server is installed
kubectl get pods -n kube-system | grep metrics-server

# Install Metrics Server if needed
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# View resource usage
kubectl top pods -l app=nginx-demo

# Check persistent volume claims
kubectl get pvc -A
```

In Kind we may need to tell the metrics server we are not using certificates or TLS for the connections.  

add this to the deployment look for the args section in the deployment and add this
```bash
- --kubelet-insecure-tls
```

**What these commands do:**
- First check if the Metrics Server is installed in your cluster
- If it's not found, install it using the official manifest
- `kubectl top` shows resource usage, similar to the 'top' command in Linux
- The last command checks for Persistent Volume Claims (PVCs), which we'll learn about next

**Expected Outcome**: You should see CPU and memory usage for each pod, measured in cores (or millicores) and bytes of memory.

## 4. Persistent Volume Claim (PVC) Testing

### What is Persistent Storage?
By default, any data stored inside a pod is lost when the pod restarts. Persistent Volumes (PVs) and Persistent Volume Claims (PVCs) solve this by providing permanent storage that survives pod restarts - think of it like an external hard drive for your pods.

### Goal: Test persistence with PVC

Create `test-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce  # This means the volume can be mounted as read-write by a single node
  resources:
    requests:
      storage: 100Mi  # Requesting 100 megabytes of storage
```

Create `test-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox  # A lightweight Linux distribution
    command: ["sleep", "3600"]  # Keep the container running for 1 hour
    volumeMounts:
    - mountPath: "/mnt/data"  # Where the storage will appear in the container
      name: test-volume
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: test-pvc  # Reference to the PVC we created
```

Test persistence:
```bash
# Apply manifests
kubectl apply -f test-pvc.yaml
kubectl apply -f test-pod.yaml

# Test data persistence
kubectl exec -it test-pod -- /bin/sh
echo "Hello, PVC!" > /mnt/data/testfile.txt
exit
kubectl delete pod test-pod
kubectl apply -f test-pod.yaml
kubectl exec -it test-pod -- cat /mnt/data/testfile.txt
```

**What these commands do:**
- Create a PVC requesting storage from the cluster
- Create a pod that uses this storage
- Write a test file to the persistent storage
- Delete and recreate the pod to prove the data survives
- Check that the file is still there

**Expected Outcome**: The test file remains available even after the pod is deleted and recreated, demonstrating persistent storage.

## 5. Networking & Service Discovery

### What is Service Discovery?
In Kubernetes, pods come and go frequently. Services provide a stable "address" that other pods can use to find and connect to your application, even as the actual pods change. Think of it like a phone number that forwards to whoever is currently "on call."

### Goal: Verify inter-pod communication and DNS resolution

```bash
# Create a service for our NGINX deployment
kubectl expose deployment nginx-demo --port=80 --type=ClusterIP

# Check service configuration
kubectl get svc

# Create a test pod to verify connectivity
kubectl run test-pod --image=busybox --rm -it -- /bin/sh

# From within the test pod, try these commands:
wget -qO- http://nginx-demo
nslookup nginx-demo
```

**What these commands do:**
- Create a service that makes our NGINX web server accessible within the cluster
- List all services in the current namespace
- Create a temporary pod for testing
- Try to reach the NGINX service and verify DNS resolution

**Expected Outcome**: The test pod should be able to connect to NGINX and see the default welcome page. The DNS lookup should resolve the service name to an IP address.

## 6. Scaling

### What is Horizontal Scaling?
Horizontal scaling means adding more pods to handle increased load, like opening more checkout lines at a busy store. Kubernetes makes this easy with a single command.

### Goal: Test horizontal scaling

```bash
# Scale deployment
kubectl scale deployment nginx-demo --replicas=5

# Verify scaling
kubectl get pods -l app=nginx-demo

# Watch the distribution of traffic
kubectl run load-test --image=busybox --rm -it -- /bin/sh
while true; do wget -qO- http://nginx-demo; sleep 1; done
```

**What these commands do:**
- Tell Kubernetes to increase the number of NGINX pods to 5
- Check that the new pods are being created
- Create a simple load test to see requests being distributed

**Expected Outcome**: You'll see Kubernetes create new pods until it reaches the desired count of 5. The service automatically starts sending traffic to all pods once they're ready.

## Troubleshooting Guide

### Common Issues and How to Debug Them:

- **Pod Issues**: 
  - Check pod logs: `kubectl logs <pod-name>`
  - Get detailed pod info: `kubectl describe pod <pod-name>`
  - Common problems: Missing images, configuration errors, resource constraints

- **PVC Issues**: 
  - Verify StorageClass configuration
  - Check for WaitForFirstConsumer binding mode with local-path-provisioner
  - Common problems: Storage class not found, insufficient resources

- **Service Issues**: 
  - Verify service targetPort matches container port
  - Check endpoint configuration
  - Common problems: Label selector mismatches, port misconfigurations

### Common Error Messages and Solutions:

1. **ImagePullBackOff**:
   - Issue: Kubernetes can't download the container image
   - Solution: Check image name, ensure proper credentials if private registry

2. **CrashLoopBackOff**:
   - Issue: Pod keeps crashing and restarting
   - Solution: Check logs with `kubectl logs`, verify container configuration

3. **Pending**:
   - Issue: Pod can't be scheduled to a node
   - Solution: Check resource constraints, node conditions

## Cleanup

After completing the tutorial, clean up your resources:

```bash
# Delete the NGINX deployment and service
kubectl delete deployment nginx-demo
kubectl delete service nginx-demo

# Delete test resources
kubectl delete -f test-pod.yaml
kubectl delete -f test-pvc.yaml
```

## Next Steps

After mastering these basics, you can explore:

4. **Node Failure Testing**: Practice node failure scenarios and pod rescheduling
5. **DNS Debugging**: Learn CoreDNS troubleshooting
6. **Security Implementation**: 
   - Set up RBAC (Role-Based Access Control)
   - Configure service accounts
   - Implement role bindings

## Best Practices

7. **High Availability**: Always maintain multiple replicas for critical services to ensure availability even if some pods fail
8. **Resource Management**: Set appropriate resource requests and limits to prevent pods from starving or overwhelming your nodes
9. **Data Protection**: Use persistent storage for any data that needs to survive pod restarts
10. **Monitoring**: Implement regular health checks and monitoring to catch issues before they affect users
11. **Security**: Follow the principle of least privilege when setting up access controls

## Common Mistakes to Avoid

12. Not setting resource limits on containers
13. Forgetting to use persistent storage for stateful applications
14. Using the default namespace for everything
15. Not labeling resources properly
16. Ignoring pod logs and events when troubleshooting

Remember: Kubernetes is complex, and it's okay to take time understanding each concept. Start with simple deployments and gradually explore more advanced features as you become comfortable with the basics.

## Additional Resources

- Official Kubernetes Documentation: https://kubernetes.io/docs/
- Kubernetes Interactive Tutorial: https://kubernetes.io/docs/tutorials/kubernetes-basics/
- NGINX Documentation: https://nginx.org/en/docs/

## Glossary

- **Pod**: Smallest deployable unit in Kubernetes
- **Node**: A worker machine in Kubernetes
- **Deployment**: Manages a replicated application
- **Service**: An abstract way to expose an application running on a set of Pods
- **PersistentVolume**: A piece of storage in the cluster
- **ReplicaSet**: Ensures that a specified number of pod replicas are running at any given time
- **Namespace**: A way to divide cluster resources between multiple users
- **kubectl**: The command-line tool for interacting with Kubernetes

## Feedback and Help

If you encounter any issues while following this tutorial:
17. Check the troubleshooting guide above
18. Review the logs using `kubectl logs`
19. Use `kubectl describe` to get detailed information about resources
20. Consult the official Kubernetes documentation
21. Reach out to your instructor for assistance