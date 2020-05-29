# Rancher Cluster

This is where configurations and scripts will go to house the cluster to host rancher will go.
Rancher will be set up on a K3s cluster.

Custom rancher installation on a K3s cluster. These instructions are adapted from [https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/).

This setup assumes you will use the following:

* The load balancer and data store are on separate instances
* You will use postgres as the backend datastore
* You are installing the newest stable version (not alpha or latest) of Rancher

## Set Up Infrastructure

Requirements:

* 2 linux nodes
* an external database. Rancher suggests MySQL
* A load balancer do direct traffic between the two nodes
* A DNS record to amp a URL to the load balancer. This is the rancher server URL and downstream
    kubernetes clusters will need to reach it

### 1. Set up linux nodes

I personally set up 4 of these nodes. These are the specs of them:

* OS: Ubuntu 18.04
* vCPUs: 2
* RAM: 4GB
* Disk: 64 GB
* Required ports:

| direction | protocol | port | nodes        |
| --------- | -------- | ----:| ------------ |
| inbound   | tcp      | 80   | K3s node     |
| inbound   | tcp      | 443  | K3s node     |
| outbound  | tcp      | 22   | K3s node     |
| outbound  | tcp      | 443  | K3s node     |
| outbound  | tcp      | 2376 | K3s node     |
| outbound  | tcp      | 6443 | K3s node     |
| inbound   | tcp      | 5432 | DataStore    |
| inbound   | tcp      | 80   | LoadBalancer |
| inbound   | tcp      | 443  | LoadBalancer |

**Ensure your nodes have static IPs.**

Install docker on all four of the linux nodes:

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Two of those nodes will be K3s nodes, one will be for the external datastore, and the other will be
an nginx load balancer.

### 2. Set up External Datastore

Copy the docker-compose file to the DataStore node, then run the following commands on it:

```bash
mkdir ./postgres-data
docker-compose up -d
```

### 3. Set up load balancer

Edit the nginx.conf file to update the ip addresses of the K3s nodes. Then, copy the nginx.conf and
docker-compose file to the LoadBalancer node, then run the following commands on it:

```bash
docker-compose up -d
```

### 4. Set up a DNS Record

Setup a DNS record in whatever DNS server you use. This DNS record needs to point to the ip of the
LoadBalancer node. Make sure this is the hostname you intend rancher to respond on.

## Set up a Kubernetes Cluster

### 1. Set up the K3s server

On each of the K3s nodes, run the following command. Make sure to update the datastore-endpoint
connection string. The values you should change are `password` and `hostname`.

```bash
curl -sfL https://get.k3s.io | sh -s - server --datastore-endpoint="postgres://postgres:password@hostname:5432/k3s"
```

### 2. Confirm K3s is running

Confirm K3s has been setup correctly. Run the following on either of the K3s nodes:

```bash
sudo k3s kubectl get nodes
```

You should see output similar to the following:

```bash
NAME               STATUS   ROLES    AGE    VERSION
ip-172-31-60-194   Ready    master   44m    v1.17.2+k3s1
ip-172-31-63-88    Ready    master   6m8s   v1.17.2+k3s1
```

Now, test the health of the cluster pods:

```bash
sudo k3s kubectl get pods --all-namespaces
```

### 3. Save and stqrt using the kubeconfig file

1. Install kubectl on your local machine.
2. Copy the file from `/etc/rancher/k3s/k3s.yaml` (on one of the K3s nodes) to `~/.kube/config` (on your local machine).
3. Update the kubeconfig file. Change the `server` directive to point to the DNS record of your load
   balancer, at port 6443.
4. You can now manage the K3s cluster using kubectl. If you have more than one kubeconfig, you can
   specify which one to use with the following command:

```bash
kubectl --kubeconfig ~/.kube/config/k3s.yaml get pods --all-namespaces
```

### 4. Check the health of your cluster pods

On your local machine, run the following command.
```bash
sudo kubectl get pods --all-namespaces
```

You should see output similar to:
```bash
NAMESPACE       NAME                                      READY   STATUS    RESTARTS   AGE
kube-system     metrics-server-6d684c7b5-bw59k            1/1     Running   0          8d
kube-system     local-path-provisioner-58fb86bdfd-fmkvd   1/1     Running   0          8d
kube-system     coredns-d798c9dd-ljjnf                    1/1     Running   0          8d
```

## Install Rancher on the Kubernetes Cluster

### 1. Install the required CLI tools

Ensure the following tools are installed and in your `$PATH`:

* kubectl
* helm (version 3) [instructions](https://docs.helm.sh/using_helm/#installing-helm)

### 2. Add the Helm chart repository

Use `helm repo add` to install the helm chart.

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
```

### 3. Create a namespace for Rancher

This namespace is where the resources created by the Helm chart will be. This
namespace should always be `cattle-system`.

```bash
kubectl create namespace cattle-system
```

### 4. Choose your SSL configuration

Your SSL configuration options are:

* Rancher-generated TLS certs
* Let's Encrypt
* Bring Your Own Certificate

These steps assume you are will use a Rancher-generated TLS cert. Note, the Helm chart option is
`ingress.tls.source=rancher`, and you will need to install cert-manager

### 5. Install cert-manager (optional based on SSL configuration)

Install cert-manager.

```bash
# Install the CustomResourceDefinition resources separately
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.0/cert-manager.crds.yaml

# **Important:**
# If you are running Kubernetes v1.15 or below, you
# will need to add the `--validate=false` flag to your
# kubectl apply command, or else you will receive a
# validation error relating to the
# x-kubernetes-preserve-unknown-fields field in
# cert-manager’s CustomResourceDefinition resources.
# This is a benign error and occurs due to the way kubectl
# performs resource validation.

# Create the namespace for cert-manager
kubectl create namespace cert-manager

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.15.0
```

Verify cert-manager is deployed correctly:

```bash
kubectl get pods --namespace cert-manager
```

You should see output similar to:
```bash
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

### 6. Install Rancher with Helm and your chosen certificate option

Rancher is the default option for `ingress.tls.source`, so you don't actually need to specify it.
Make sure to update `hostname` to the DNS name you pointed to your load balancer.

```bash
helm install rancher rancher-stable/rancher --namespace cattle-system --set hostname=CHANGEME
```

Wait for rancher to be rolled out:

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

The output will look like:

```bash
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```


### 7. Verify that the rancher server is successfully deployed

After adding the secrets, check if Rancher was rolled out successfully:

```bash
kubectl -n cattle-system rollout status deploy/rancher
```

The output will look like:

```bash
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```

If you see the following error: `error: deployment "rancher" exceeded its progress deadline`, you can check the status by running the following command:

```bash
kubectl -n cattle-system get deploy rancher
```

The output will look like:

```bash
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rancher   3         3         3            3           3m
```

It should show the same count for `DESIRED` and `AVAILABLE`.


### 8. Save your options

Make sure you save the `--set` options you used in step 6. If you upgrade rancher with Helm you will need them.

