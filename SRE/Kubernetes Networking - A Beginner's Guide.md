## Introduction

Welcome to Kubernetes Networking! This guide will help you understand how applications communicate in Kubernetes. Don't worry if you're new to this - we'll explain everything step by step.

Think of Kubernetes networking like a modern office building:
- Each application (pod) has its own office (IP address)
- There's a directory (DNS) to help find services
- There are reception desks (Services) that route visitors
- Some offices are private (ClusterIP), while others have public entrances (LoadBalancer)

## Prerequisites

Before we start, you'll need:
- A Kubernetes cluster (we'll use KIND - Kubernetes IN Docker)
- kubectl (the Kubernetes command-line tool)
- Docker installed
- golang installed (for some advanced features)

If you're missing any of these, ask your instructor for help getting set up.

## Part 1: Pod-to-Pod Communication

### Understanding Pods and Their Network

Just like people need addresses to receive mail, every pod in Kubernetes needs an address to receive network traffic. Here's how it works:

- Every pod gets its own IP address
- Pods can talk directly to any other pod
- All pods can find each other automatically
- No complicated setup needed - Kubernetes handles it!

### Lab 1: Your First Pod Communication Test

Let's create two pods and make them talk to each other:

```yaml
# save as pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: nginx
    image: nginx    # This pod runs a web server
---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: busybox
    image: busybox    # This pod will be our test pod
    command: ['sh', '-c', 'while true; do sleep 3600; done']
```

Let's try it out:

```bash
# Create our pods
kubectl apply -f pods.yaml

# Wait for both pods to be ready
kubectl wait --for=condition=Ready pod/pod1 pod/pod2

# Find pod1's address
POD1_IP=$(kubectl get pod pod1 -o jsonpath='{.status.podIP}')

# Send a message from pod2 to pod1
kubectl exec pod2 -- wget -O- http://$POD1_IP
```

**What's Happening Here?**
1. We created a web server pod (pod1)
2. We created a test pod (pod2)
3. We found pod1's address
4. We used pod2 to send a message to pod1
5. If successful, you'll see the NGINX welcome page

## Part 2: Understanding Services

### Why Do We Need Services?

Imagine if your favorite restaurant changed its phone number every day - that would be frustrating! Similarly, pods in Kubernetes can come and go, getting new IP addresses each time. Services solve this by providing a stable "phone number" that never changes.

### Types of Services

6. **ClusterIP**
   - Internal access only
   - Like an office building's internal phone system
   - Perfect for services that only need to be accessed by other pods

7. **NodePort**
   - Opens a port on every node
   - Like adding a public entrance to every side of the building
   - Good for development and testing

8. **LoadBalancer**
   - Creates an external load balancer
   - Like having a smart reception desk that directs visitors
   - Best for production applications

9. **ExternalName**
   - Points to an external service
   - Like having a business card that redirects to another company
   - Used for integrating external services

### Lab 2: Creating Different Types of Services

#### 2.1: Internal Service (ClusterIP)

```yaml
# save as clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Try it out:
```bash
# Create the service and deployment
kubectl apply -f clusterip.yaml

# Test internal access
kubectl run test-pod --image=busybox -it --rm -- wget -O- http://nginx-clusterip
```

**What's Happening?**
- We created a service that manages traffic to our NGINX pods
- The service finds pods with the label "app: nginx"
- We can access our pods using the service name "nginx-clusterip"

#### 2.2: NodePort Service

```yaml
# save as nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080  # This port will be opened on every node
```

Try it:
```bash
# Create the NodePort service
kubectl apply -f nodeport.yaml

# Find your node's IP address
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# Access the service
curl http://$NODE_IP:30080
```

**What's Happening?**
- We created a service that opens port 30080 on every node
- Anyone can access our application through any node's IP address
- Great for testing, but not recommended for production

## Part 3: Load Balancing

### Understanding Load Balancers

Think of a load balancer like a smart receptionist who makes sure visitors are evenly distributed among available staff members. In Kubernetes, load balancers:
- Distribute traffic across pods
- Provide a single entry point
- Can handle external traffic

### Lab 3: Setting Up Load Balancing

First, let's set up our cloud provider (we'll use KIND's built-in provider):

```bash
# Install Cloud Provider KIND
go install sigs.k8s.io/cloud-provider-kind@latest

# Start the cloud provider (in a separate terminal)
sudo cloud-provider-kind
```

Now create a load-balanced service:

```yaml
# save as loadbalancer.yaml
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: http-echo
spec:
  containers:
  - name: foo-app
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    command:
    - /agnhost
    - serve-hostname
    - --http=true
    - --port=8080
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: http-echo
spec:
  containers:
  - name: bar-app
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    command:
    - /agnhost
    - serve-hostname
    - --http=true
    - --port=8080
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  type: LoadBalancer
  selector:
    app: http-echo
  ports:
  - port: 5678
    targetPort: 8080
```

Try it out:
```bash
# Apply the configuration
kubectl apply -f loadbalancer.yaml

# Wait for external IP
kubectl wait --for=jsonpath='{.status.loadBalancer.ingress[0].ip}' service/foo-service

# Get LoadBalancer IP
LB_IP=$(kubectl get svc/foo-service -o=jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test load balancing
for _ in {1..10}; do
  curl ${LB_IP}:5678
done
```

**What's Happening?**
- We created two pods that return their hostname
- We created a LoadBalancer service to distribute traffic between them
- The loop sends multiple requests to see the traffic being distributed

## Part 4: Network Policies

### Understanding Network Policies

Network policies are like security guards who control which pods can talk to each other. By default, all pods can communicate with any other pod, but sometimes you want to restrict this.

### Lab 4: Creating a Network Policy

```yaml
# save as network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          access: allowed
    ports:
    - protocol: TCP
      port: 80
```

Try it:
```bash
# Apply the policy
kubectl apply -f network-policy.yaml

# Test access (should fail)
kubectl run test-pod --image=busybox -it --rm -- wget -O- --timeout=5 http://nginx-clusterip

# Test access with allowed label (should succeed)
kubectl run test-pod --labels="access=allowed" --image=busybox -it --rm -- wget -O- http://nginx-clusterip
```

**What's Happening?**
- We created a policy that only allows pods with label "access: allowed" to reach our NGINX pods
- The first test fails because the pod doesn't have the required label
- The second test succeeds because we added the correct label

## Troubleshooting Guide

### Common Problems and Solutions

10. **"I can't access my service!"**
   ```bash
   # Check if your service exists
   kubectl get service my-service
   
   # Check if it's connected to pods
   kubectl describe service my-service
   ```
   Look for:
   - Are there any endpoints listed?
   - Do the pod labels match the service selector?

11. **"My LoadBalancer doesn't get an IP!"**
   ```bash
   # Check service events
   kubectl describe service my-service
   ```
   Common causes:
   - Cloud provider not configured
   - Resource limitations
   - Network issues

12. **"Pods can't talk to each other!"**
   ```bash
   # Check network policies
   kubectl get networkpolicies
   
   # Test basic connectivity
   kubectl run test-pod --image=busybox -it --rm -- ping <pod-ip>
   ```

### Debugging Steps
13. Start simple: Can pods reach themselves?
14. Check service configuration
15. Verify network policies
16. Look at logs and events

## Best Practices

17. **Service Design**
   - Use meaningful labels
   - Document your port choices
   - Consider security implications

18. **Load Balancing**
   - Set up health checks
   - Monitor load balancer status
   - Use appropriate service types

19. **Network Policies**
   - Start restrictive and open as needed
   - Document your policies
   - Test thoroughly

## Cleanup

Don't forget to clean up after your labs:

```bash
# Clean up services
kubectl delete service nginx-clusterip nginx-nodeport foo-service

# Clean up deployments
kubectl delete deployment nginx-deployment

# Clean up pods
kubectl delete pod foo-app bar-app

# Clean up network policies
kubectl delete networkpolicy access-nginx
```

## Additional Resources

20. Official Documentation:
   - [Kubernetes Services Guide](https://kubernetes.io/docs/concepts/services-networking/service/)
   - [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

21. Tools and References:
   - [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
   - [KIND Documentation](https://kind.sigs.k8s.io/docs/user/configuration/)

## Remember

- Networking can be complex - take it step by step
- Always start debugging from the simplest component
- Don't hesitate to ask for help when stuck
- Practice is key to understanding these concepts

Good luck with your Kubernetes networking journey!