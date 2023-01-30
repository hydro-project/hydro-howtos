# Avoiding Kubernetes Preemption During Experiments

It’s helpful to think of Kubernetes as a declarative, convergence-based scheduling system. Kubernetes deployments (which we use as the basis for Hydroplane processes in GKE and EKS) are responsible for declaring the desired state of the deployment (including, most importantly for our purposes, how many pods the deployment is running), and the Kubernetes control plane is responsible for transitioning the cluster to that desired state in a controlled way.

When Kubernetes isn’t under resource pressure, its nodes are online and the population of nodes is stable, and the pods it’s running aren’t crashing, pods don’t tend to move around that much. When the cluster’s resources start to become constrained, existing nodes disappear, new nodes appear, or pods crash, the Kubernetes control plane is entitled to kill pods (by sending them SIGTERM and, after a delay, SIGKILL) and move them around as it sees fit.

In general, I don’t know of any way to prevent pre-emption or eviction from happening 100% of the time. However, there are things you can do to make pre-emption and eviction extremely unlikely.

## Specify Resource Requests and Limits

When Kubernetes creates a pod, it applies a [quality-of-service class](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/) to it. Pods are scheduled with different priorities depending on their quality-of-service class.

For a pod to be in the “Guaranteed” QoS class, it has to have both CPU and memory requests and limits specified, and the amount requested for both CPU and memory has to match the limit. Because Kubernetes knows how much of each resource the pod will consume, it will schedule pods with the “Guaranteed” QoS first, and won’t pre-empt them unless the cluster runs out of resources and critical pods need to be scheduled instead.

Playing with resource requests and limits is one way to get pods where you want them in your cluster, but it’s far from the most surefire way. Regardless, making sure your pods are in the “Guaranteed” QoS class is something that’s never a bad idea.

Hydroplane allows you to set CPU and memory requests and limits when creating a process. Here’s an example process that sets its memory limit to 0.25 vCPUs and its memory limit to 256 MiB.

```json
{
 "process_name": "nginx",
 "container": {
   "image_uri": "nginxdemos/hello",
   "ports": [
     {
       "container_port": 80,
       "host_port": 8080,
       "protocol": "tcp"
     }
   ],
   "resource_request": {
     "cpu_vcpu": "0.25",
     "memory_mib": 256
   },
   "resource_limit": {
     "cpu_vcpu": "0.25",
     "memory_mib": 256
   }
 },
 "has_public_ip": true
}
```

## Over-Provision your Node Pool and/or Disable Auto-Scaling

The most common driver of Kubernetes’ eviction and preemption of pods is resource pressure. An easy way to limit that pressure while still giving Kubernetes’ scheduler room to act is to over-provision Kubernetes’ node pool so that you know you have enough resource headroom to accommodate all processes in the experiment.

In GKE, you can’t control your node pool in Autopilot clusters; you can only do that in Standard clusters. In Standard clusters, you can choose to turn auto-scaling on or off, control the minimum and maximum number of nodes in the pool, and set other kinds of autoscaling behavior. [This page from the Google Cloud docs](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-autoscaler) has a lot of information about all your various autoscaling options.

## Force Your Processes onto Nodes with Node Selectors

Kubernetes has a facility called [node selectors](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) that can constrain the set of nodes that a pod is allowed to be scheduled onto. This constraint is based on the node’s labels. You can tell what labels a node has by looking at the Google Cloud console, or through kubectl:

```bash
gcloud container clusters get-credentials --region <cluster region> <cluster-name>
kubectl get nodes --show-labels
```

Each node comes with a lot of labels that describe what hardware and operating system it’s running, its hostname, and a whole bunch of other things.

Hydroplane allows processes to specify a set of node selectors that constrain where they run. The [`nginx-advanced.json` example](https://github.com/hydro-project/hydroplane/blob/main/examples/nginx-advanced.json) in the Hydroplane repo constrains the nginx process to run on a node with the label `kubernetes.io/hostname=gke-hydroplane-test-clus-default-pool-ffab0dea-gmz0`

If you’re looking to auto-scale and need a “warm pool” of nodes ready to accept new processes quickly, make a [node pool](https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools) for those processes and use node selectors to assign your processes to that new pool. In GKE, every node has a `cloud.google.com/gke-nodepool` label whose value is that node’s pool, so specifying a node selector `cloud.google.com/gke-nodepool=my-pool-name` would constrain the process to only run on nodes in that node pool.

You can also provide custom labels for individual nodes (or entire node pools) yourself, and use node selectors to restrict processes to running in those nodes. This might help make it easier for you to run experiments reproducibly over time.

There’s a more general-purpose version of node selectors called [taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/), but it’s probably more sophisticated than you’re likely to need, so I stuck with implementing node selectors.

## When to Use What

When in doubt, assign resource requests and limits to your processes.

When you absolutely, positively have to be sure that your processes won’t move around, and you’ve got a static population of them, use node selectors to constrain each process to a given node.

Otherwise, some combination of node selectors and node pools that are provisioned with some headroom will go a long way toward making sure your processes stay running in the same place reliably.
