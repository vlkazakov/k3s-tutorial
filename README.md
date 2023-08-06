# K3s Detailed Installation and Configuration Tutorial

## Table of Contents

- [Chapter 1: Introduction to k3s](#chapter-1-introduction-to-k3s)
- [Chapter 2: Installing k3s for Production](#chapter-2-installing-k3s-for-production)  
- [Chapter 3: Configuring k3s](#chapter-3-configuring-k3s)
- [Chapter 4: Managing k3s Clusters](#chapter-4-managing-k3s-clusters)
- [Chapter 5: Deploying Applications on k3s](#chapter-5-deploying-applications-on-k3s)
- [Chapter 6: Conclusion](#chapter-6-conclusion)

## Chapter 1: Introduction to k3s

K3s (pronounced "kiss") is a lightweight Kubernetes distribution developed by Rancher Labs as a certified Kubernetes conformant platform. It is optimized for edge computing use cases such as IoT devices, Rasberry Pis, edge servers, and other resource-constrained environments.

Some key advantages of k3s are:

- Easy to install - Installs in less than a minute with a single binary file less than 40MB in size. Does not require an external database.

- Lightweight - Built for ARM and x86 architectures. Optimized storage backend uses SQLite instead of etcd. Runs on nodes with just 512MB RAM and 200MB disk space. 

- Security focused - Enables mTLS authentication between components by default. Minimal attack surface area.

- Operationally simple - Single binary installation. Upgrades are easy rollouts. No node-reboot required. 

- Cloud native and Kubernetes conformant - Certified conformant with upstream Kubernetes releases. Native support for Helm, Prometheus, Traefik etc.

In this manual, we will learn how to setup, operate and manage production-grade k3s clusters. We will cover topics like installation, configuration, cluster management, deployment of applications and more.

By the end of the manual, you should be able to:

- Install single and multi-node k3s clusters

- Secure cluster access with proper authentication

- Monitor and manage clusters using native Kubernetes tooling

- Deploy sample applications via kubectl and Helm

- Perform rolling updates and cluster upgrades

- Troubleshoot common issues

The manual is intended for beginners who want to get started with Kubernetes, as well as experienced admins looking to build lightweight clusters. Familarity with Linux, Docker and networking fundamentals will be helpful.

Let's get started by first installing a single node k3s cluster on a Linux machine. We will use the installation script method in this example.

First, update the package index on your Linux machine:

```
sudo apt update
```

Next, run the installation script provided by k3s to install the single binary:

```
curl -sfL https://get.k3s.io | sh -
```

This will download the latest release, install it to /usr/local/bin/k3s and run the server.

Verify that k3s is up and running:

```
sudo k3s kubectl get nodes
```

This should print the node details. The server may take a few seconds to come up. 

That's it! We now have a lightweight Kubernetes cluster running with a single command. In the next chapter we will look at configuration options and installing in production setups.

## Chapter 2: Installing k3s for Production

In the previous chapter, we did a simple single node installation of k3s using the convenience script. However, for production deployments, we need to use the binary or automated configuration management tools like Ansible.

The k3s binary can be directly downloaded from GitHub releases and installed. For example:

```
curl -LO https://github.com/k3s-io/k3s/releases/download/v1.23.4+k3s1/k3s
chmod +x k3s
sudo mv k3s /usr/local/bin/
```

This downloads a specific version rather than latest. You can then run the binary to start k3s: 

```
sudo k3s server --disable traefik
```

We disabled the built-in Traefik load balancer here since we won't be exposing services externally.

For automated cluster setup, using tools like Ansible is recommended. k3s provides an Ansible role which handles downloading the binary as well as generating the config file. 

A sample playbook would look like:

```yaml
- hosts: servers
  roles:
    - role: k3s
      k3s_release_version: v1.23.4+k3s1 
```

This will install k3s on all hosts listed under the `servers` group using the given version.

Some key configuration flags and environment variables available include:

```
--disable <component>  - Disable default components like traefik, local-storage etc.

K3S_TOKEN - Shared secret for node registration

K3S_CLUSTER_SECRET - Cluster secret for encryption 

K3S_KUBECONFIG_MODE - Control kubeconfig file permissions

K3S_NODE_NAME - Override hostname with custom node name
```

For a highly available cluster, we should run the k3s server on multiple nodes with the `--server https://node1:6443` flag pointing to the other server URLs. 

The database outer data stored by k3s can be externalized by passing `--datastore-endpoint` to stand up a separate database like PostgreSQL or MySQL.

In the next chapter we will look at k3s configuration files, access control, networking and other cluster settings.

## Chapter 3: Configuring k3s 

In this chapter we will explore the main configuration files and settings that control access, security, networking and other aspects of a k3s cluster.

### Configuration Files

The key configuration files used by k3s are:

- /var/lib/rancher/k3s/server/db/state.yaml - Internal state for cluster data

- /etc/rancher/k3s/config.yaml - Main config file for components

- /etc/rancher/k3s/kubeconfig.yaml - Kubeconfig for accessing cluster

- /etc/systemd/system/k3s.service - Systemd unit file

The config.yaml file controlsparameters like the embedded datastore, networking, registries etc. For example:

```yaml
disable:
  - traefik

cluster_cidr: 10.42.0.0/16 

service_cidr: 10.43.0.0/16
```

This disables traefik and configures pod and service CIDRs.

### Access Control 

By default k3s does not come with authentication enabled. We need to secure access with proper credentials. 

One option is to pass `K3S_TOKEN` environment variable with a shared secret when installing nodes.

For user access, delegated permissions can be granted via RBAC rules. For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view  
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: john
```

This grants read-only view access to the `john` user. Access requires a valid kubeconfig.

### Networking

The network range for pods and services can be configured via the `cluster_cidr` and `service_cidr` options. 

CNI plugins like Flannel are used for overlay networking. The CNI can be customized by passing:

```
--flannel-backend=none --cni-config=/etc/cni/net.d
```

And placing custom CNI manifests under the `/etc/cni/net.d` directory.

That covers the basics of k3s configuration and cluster access. Up next we will look at managing and operating a cluster.

## Chapter 4: Managing k3s Clusters 

Now that we have k3s installed and configured, let's look at managing cluster operations like adding nodes, upgrading, monitoring, troubleshooting etc.

### Interacting with the Cluster

The main tool for interacting with the cluster is kubectl. The installation script automatically configures kubectl to point to the cluster. 

Some common operations are:

```
kubectl get nodes        # List nodes
kubectl get pods         # List pods  
kubectl describe node    # Get node status
kubectl logs mypod       # View pod logs
kubectl exec -it shell   # Start interactive shell inside pod
```

### Adding Worker Nodes

Additional nodes can join the cluster by running the k3s agent and passing the server URL:

```
curl -sfL https://get.k3s.io | K3S_URL=https://node1:6443 K3S_TOKEN=mynodetoken sh -
```

Get the token from /var/lib/rancher/k3s/server/node-token on the server. 

The new node should register itself and be available to schedule pods.

### Upgrading the Cluster 

k3s makes upgrades easy by simply replacing the server binary and restarting the service:

```
curl -LO https://github.com/k3s-io/k3s/releases/download/v1.24.0+k3s1/k3s
sudo systemctl restart k3s
```

Nodes will be automatically updated to match the new server version.

### Monitoring and Debugging

The inbuilt metrics server can be used for basic monitoring of resources, pods, nodes etc.  

For richer metrics collection, deploy Prometheus-operator via Helm.

Common issues like node unreachability can be debugged by checking logs:

```
journalctl -u k3s
```

In the next chapter we will deploy sample applications on k3s to test out the cluster.

## Chapter 5: Deploying Applications on k3s

Now that we have a production-grade k3s cluster running, let's deploy some sample applications to test it out.

### Kubernetes Basics

Kubernetes provides abstractions like pods, deployments, services and ingress to deploy containerized applications.

Some key concepts:

- Pods - Smallest units which encapsulate one or more containers

- Deployments - Declarative way to rollout pod replicas

- Services - Logical set of pods with stable networking

- Ingress - Route external traffic to services

We define these resources via YAML manifests. For example:

```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx  
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
        image: nginx
        
---

apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

This will deploy 3 nginx pods fronted by a load balancing service.

### Sample Applications

Let's deploy some common applications:

1. Web UI - Deploy a React app as a Docker image via a deployment and service

2. MySQL - Use a Helm chart to install MySQL with persistent volume

3. Redis - Create a deployment and clusterIP service 

4. RabbitMQ - Leverage HostPath to persist queue data

For ingress, we can set up the NGINX ingress controller to route traffic from outside to services inside the cluster.

Overall, k3s provides a robust platform to run Dockerized apps with Kubernetes native deployment options.

In the final chapter, we will summarize key learnings and next steps.

## Chapter 6: Conclusion 

Let's recap what we learned about k3s and discuss next steps:

### Key Takeways

- k3s is a lightweight Kubernetes distribution focused on edge/IoT use cases

- It can be installed on a single node for evaluation or clustered for production

- The configuration file, kubeconfig and binaries are the core components

- Access control should be implemented with tokens, RBAC policies etc. 

- Networking, registries, storage can be customized via config options

- The cluster can be monitored, upgraded and troubleshot using kubectl

- Common applications can be deployed as pods, deployments and services

With these fundamentals, you should be able to get started running containerized workloads on k3s clusters.

### What's Next?

Here are some recommendations for taking your k3s skills to the next level:

- Try running k3s on ARM/Raspberry Pi to see performance

- Integrate with hardware sensors on edge devices 

- Deploy CNI plugins like Calico for advanced networking

- Evaluate k3s as a Kubernetes distribution for your production environment

- Dig deeper into Kubernetes concepts like jobs, volumes, Helm etc.

- Explore the k3s architecture - how the embedded database, agents etc work

- Participate in the k3s community forums and GitHub project

Thank you for following along with this comprehensive k3s manual. Reach out to me if you need any clarification or have suggestions to improve it. I hope you found this tutorial helpful. Happy k3sing!
