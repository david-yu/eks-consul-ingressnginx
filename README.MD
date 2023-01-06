# NGINX ingress controller integration with Consul on Kubernetes using Transparent Proxy 

## Requirements
- Kubernetes cluster on any Cloud Provider
- `kubectl` installed locally
- `helm` installed locally

## Cloud Provider Setup

Consul on K8s can be deployed on any K8s distro such as EKS, GKE, and AKS. 

### AWS Setup 

#### Requirements 

- An AWS account and a region that support EKS
- Environment variables to access AWS account locally
- `eksctl` installed locally

#### Steps

1. Create a EKS cluster:
  ```bash
  eksctl create cluster --name=<cluster name> --region=<region> --nodes=3 
  ```
  
2. Import kubectl config
  ```bash
  aws eks update-kubeconfig --region <region> --name <cluster name>
  ```

### GKE Setup

#### Requirements
- A Google account and a region that support GKE
- Environment variables to access GKE account locally
- `gcloud` installed locally

#### Steps

1. Set environment variables
  ```
  export PROJECT=<PROJECT ID>
  gcloud config set project $PROJECT
  gcloud config set compute/zone us-west1-c
  ```
  
2. Create a GKE Cluster:
  ```
  gcloud container clusters create nginx-consulk8s --num-nodes=3 --machine-type "e2-highcpu-4" --enable-autoscaling --min-nodes 1 --max-nodes 4
  ```

3. Import kubectl config
  ```
  gcloud container clusters get-credentials nginx-consulk8s
  ```

## Installation

Deploy Consul

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install consul hashicorp/consul --values consul-values.yaml --create-namespace --namespace consul
```

Add deny all intention
```bash
kubectl apply -f denyall.yaml
```

Deploy nginx ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx \                                                                                                    ─╯
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace --values nginxingress-values.yaml
```

Add configuration for Dialed Directly

```bash
kubectl apply -f sd-direct.yaml
```

Add configuration for ingress config to static-server.
```bash
kubectl apply -f ingress.yaml
```

deploy static-server
```bash
kubectl apply -f static-server.yaml
```

add intention to static server from ingress
```bash
kubectl apply -f allow-static-server.yaml
```

To get the ingress endpoint provisioned in ELB:
```bash
kubectl get service  -n ingress-nginx
```

Ensure you get back a `hello world` response:

```bash
STATIC_SERVER_LB=$(kubectl get service  -n ingress-nginx ingress-nginx-controller -o jsonpath="{.status.loadBalancer.ingress[*].hostname}")
curl $STATIC_SERVER_LB
```

