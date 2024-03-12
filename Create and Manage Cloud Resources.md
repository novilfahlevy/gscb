# Create and Manage Cloud Resources: Challenge Lab

## Set default region and zone

```bash
gcloud config set compute/region REGION
```

```bash
gcloud config set compute/zone ZONE
```

## Task 1: Create a project jumphost instance

```bash
gcloud compute instances create nucleus-jumphost-853 --machine-type=e2-micro --network nucleus-vpc
```

## Task 2: Create a Kubernetes service cluster

```bash
gcloud container clusters create nucleus-cluster --network nucleus-vpc
```

```bash
gcloud container clusters get-credentials nucleus-cluster --zone [cluster-zone]
```

```bash
kubectl create deployment hello-server --image="hello-app(gcr.io/google-samples/hello-app:2.0)"
```

```bash
kubectl expose deployment hello-server --port=8083 --type="LoadBalancer"
```

## Task 3: Set up an HTTP load balancer

### 1. Create an instance template. Don't use the default machine type. Make sure you specify e2-medium as the machine type.

```bash
gcloud compute instance-templates create nucleus-web-server-template \
   --machine-type=e2-medium \
   --network nucleus-vpc \
   --metadata=startup-script='#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i "s/nginx/Google Cloud Platform - \$HOSTNAME/" /var/www/html/index.nginx-debian.html'
```

### 2. Create a managed instance group based on the template.

```bash
gcloud compute instance-groups managed create nucleus-web-server-group \
   --template=nucleus-web-server-template \
   --size=2
```

### 3. Create a firewall rule named as Firewall rule to allow traffic (80/tcp).

```bash
gcloud compute firewall-rules create accept-tcp-rule-483 \
   --action=allow \
   --direction=ingress \
   --source-ranges=0.0.0.0/0 \
   --rules=tcp:80
```

### 4. Create a health check.

```bash
gcloud compute http-health-checks create http-basic-check --port 80
```

### 5. Create a backend service and add your instance group as the backend to the backend service group with named port (http:80).

```bash
gcloud compute instance-groups managed set-named-ports nucleus-web-server-group --named-ports http:80
```

```bash
gcloud compute backend-services create nucleus-web-server-backend \
   --protocol=HTTP \
   --port-name=http \
   --http-health-checks=http-basic-check \
   --global
```

```bash
gcloud compute backend-services add-backend nucleus-web-server-backend \
   --instance-group=nucleus-web-server-group \
   --global
```

### 6. Create a URL map, and target the HTTP proxy to route the incoming requests to the default backend service.

```bash
gcloud compute url-maps create nucleus-web-server-map --default-service nucleus-web-server-backend
```

### 7. Create a target HTTP proxy to route requests to your URL map

```bash
gcloud compute target-http-proxies create http-lb-proxy --url-map nucleus-web-server-map
```

### 8. Create a forwarding rule.

```bash
gcloud compute forwarding-rules create http-content-rule \
   --global \
   --target-http-proxy=http-lb-proxy \
   --ports=80
```

```bash
gcloud compute forwarding-rules list
```