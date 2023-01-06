# NGINX ingress (ingress-nginx) controller integration with Consul on Kubernetes using Transparent Proxy 

## Requirements
- Kubernetes cluster on any Cloud Provider
- `kubectl` installed locally
- `helm` installed locally

## Cloud Provider Setup

Consul on K8s can be deployed on any K8s distro such as EKS, GKE, and AKS. Below are examples on how to get started on AWS and Google Cloud. 

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

### Google Cloud Setup

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

1. Deploy Consul
  ```bash
  helm repo add hashicorp https://helm.releases.hashicorp.com
  helm install consul hashicorp/consul --values consul-values.yaml --create-namespace --namespace consul
  ```

2. Add deny all intention
  ```bash
  kubectl apply -f denyall.yaml
  ```

3. Deploy nginx ingress
  ```bash
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
  helm upgrade --install ingress-nginx ingress-nginx \                                                                                                    ─╯
    --repo https://kubernetes.github.io/ingress-nginx \
    --namespace ingress-nginx --create-namespace --values nginxingress-values.yaml
  ```

4. Add configuration for Dialed Directly
  ```bash
  kubectl apply -f sd-direct.yaml
  ```

5. Set NGINX load balancer IP as an environment variable:  
  ```bash
  export NGINX_INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o json | jq -r '.status.loadBalancer.ingress[].ip')
  ```

6. Generate Ingress Resource with nginx load balancer ip. 

  ```
  cat <<EOF > ingress-resource.yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: test-nginx-ingress
    annotations:
      nginx.ingress.kubernetes.io/proxy-body-size: 10G
      nginx.ingress.kubernetes.io/enable-underscores-in-headers: "true"
      nginx.ingress.kubernetes.io/proxy-read-timeout: "300"
      nginx.ingress.kubernetes.io/proxy-send-timeout: "300"
      nginx.ingress.kubernetes.io/proxy-connect-timeout: "1200"
      # nginx.ingress.kubernetes.io/client-header-timeout: "300"
      nginx.ingress.kubernetes.io/upstream-keepalive-timeout: "300"
      nginx.ingress.kubernetes.io/proxy-buffer-size: 8k
  spec:
    ingressClassName: nginx
    rules:
    - host: "$NGINX_INGRESS_IP.nip.io"
      http:
        paths:
        - path: /server
          pathType: Prefix
          backend:
            service:
              name: static-server
              port: 
                number: 8080
    defaultBackend:
      service:
        name: static-server
        port:
          number: 8080
  EOF
  ```

7. Apply configuration for ingress config to route traffic to `static-server`.
  ```bash
  kubectl apply -f ingress-resource.yaml
  ```

8. Deploy `static-server` deployment. 
  ```bash
  kubectl apply -f static-server.yaml
  ```

9. Apply intention to `static-server` from ingress
  ```bash
  kubectl apply -f allow-static-server.yaml
  ```

10. Ensure you get back a `hello world` response when routing requests to the NGINX hostname for the `static-server` route. 
  ```bash
  curl ${NGINX_INGRESS_IP}.nip.io
    "hello world"
  ```

