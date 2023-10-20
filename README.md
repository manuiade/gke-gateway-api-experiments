# GKE Gateway APIs Experiments

## Requirements

- GCP Project
- GCP User with enough IAM roles on project (Owner for testing purposes)
- Active domain and access to registrar to add CNAME records for Domain validation

## Setup

### Set env vars

```
export PROJECT_ID=<PROJECT_ID>
export DOMAIN=<SUBDOMAIN.DOMAIN>
export DOMAIN_NAME=<SUBDOMAIN_DOMAIN>

export PROJECT_NUMBER="$(gcloud projects describe $PROJECT_ID --format='get(projectNumber)')"
export NETWORK=gke-network
export SUBNETWORK=gke-subnet
export REGION=europe-west1
export ZONE=europe-west1-c
export ROUTER=gke-router
export NAT=gke-nat
export SA_GKE=gke-sa
export CLUSTER=gke-cluster
export CLUSTER_CONTROL_PLANE_CIDR=172.16.0.0/28
export CERT_MAP=store-cert-map
export IP_NAME=lb-ext-ip

gcloud config set project $PROJECT_ID
```

### Substitute variables in YAML files

```
sed -i "s/CERT_MAP/$CERT_MAP/g"  platform-admin/gateway.yaml
sed -i "s/IP_NAME/$IP_NAME/g"  platform-admin/gateway.yaml
sed -i "s/DOMAIN/$DOMAIN/g"  developer/httproute.yaml
```

### Setup networking

```
# Create VPC
gcloud compute networks create $NETWORK \
    --subnet-mode=custom \
    --mtu=1460 \
    --bgp-routing-mode=regional

# Create subnetwork
gcloud compute networks subnets create $SUBNETWORK \
    --range=10.100.0.0/20  \
    --network=$NETWORK \
    --region=$REGION

# Enable nodes access to Internet
gcloud compute routers create $ROUTER \
    --network=$NETWORK \
    --region=$REGION

gcloud compute routers nats create $NAT \
    --router $ROUTER \
    --region=$REGION \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges

# Reserve global external static IP for LB
gcloud compute addresses create $IP_NAME \
    --global
```


### Create GKE cluster with Gateway APIs enabled

```
# Dedicated cluster service account
gcloud iam service-accounts create $SA_GKE

# Create private cluster
gcloud container clusters create $CLUSTER \
  --zone $ZONE \
  --enable-private-nodes \
  --master-ipv4-cidr $CLUSTER_CONTROL_PLANE_CIDR \
  --no-enable-master-authorized-networks \
  --enable-ip-alias \
  --service-account $SA_GKE@$PROJECT_ID.iam.gserviceaccount.com \
  --num-nodes 2 \
  --spot \
  --gateway-api=standard

gcloud container clusters get-credentials $CLUSTER --zone $ZONE
```

### Setup Certificate Manager for SSL certificates (use DNS authz)

```
gcloud services enable certificatemanager.googleapis.com

gcloud certificate-manager maps create $CERT_MAP \
    --location global \
    --description "Global cert map"

gcloud certificate-manager dns-authorizations create $DOMAIN_NAME \
    --domain $DOMAIN \
    --location global

CHALLENGE_NAME=$(gcloud certificate-manager dns-authorizations describe $DOMAIN_NAME --format json | jq -r '.dnsResourceRecord.name')
CHALLENGE_DATA=$(gcloud certificate-manager dns-authorizations describe $DOMAIN_NAME --format json | jq -r '.dnsResourceRecord.data')

## IMPORTANT: add a CNAME record on your registrar with CHALLENGE_NAME and CHALLENGHE_DATA values to validate certificate

gcloud certificate-manager certificates create $DOMAIN_NAME \
    --domains $DOMAIN \
    --location global \
    --dns-authorizations $DOMAIN_NAME

gcloud certificate-manager maps entries create $DOMAIN_NAME \
    --location global \
    --map $CERT_MAP \
    --hostname $DOMAIN \
    --certificates $DOMAIN_NAME
```

### Create namespaces for gateway and application resources

```
kubectl apply -f platform-admin/namespaces.yaml
```


### Deploy GKE Gateway controller for Global HTTP/S ALB

```
kubectl apply -f platform-admin/gateway.yaml -n gateway
kubectl apply -f platform-admin/https-redirect.yaml -n gateway
```


### Deploy application and httproutes as developer

```
kubectl apply -f developer/app.yaml -n app
kubectl apply -f developer/httproute.yaml -n app
```

## Test call (after pointing an A record from LB IP to selected DOMAIN)

```
curl https://$DOMAIN
curl https://$DOMAIN -H "env: canary"
curl https://$DOMAIN/de
```

## Cleanup

### Substitute back variables in YAML files

```
sed -i "s/$CERT_MAP/CERT_MAP/g"  platform-admin/gateway.yaml
sed -i "s/$IP_NAME/IP_NAME/g"  platform-admin/gateway.yaml
sed -i "s/$DOMAIN/DOMAIN/g"  developer/httproute.yaml
```

### Delete GCP resources

```
gcloud container clusters delete $CLUSTER \
    --zone $ZONE \
    --quiet

gcloud iam service-accounts delete $SA_GKE@$PROJECT_ID.iam.gserviceaccount.com \
    --quiet

gcloud certificate-manager maps entries delete $DOMAIN_NAME \
    --location global \
    --map $CERT_MAP \
    --quiet

gcloud certificate-manager certificates delete $DOMAIN_NAME \
    --location global \
    --quiet

gcloud certificate-manager dns-authorizations delete $DOMAIN_NAME \
    --location global \
    --quiet

gcloud certificate-manager maps delete $CERT_MAP \
    --location global \
    --quiet

gcloud compute addresses delete $IP_NAME \
    --global \
    --quiet

gcloud compute routers nats delete $NAT \
    --router $ROUTER \
    --region=$REGION \
    --quiet

gcloud compute routers delete $ROUTER \
    --region=$REGION \
    --quiet

gcloud compute networks subnets delete $SUBNETWORK \
    --region=$REGION \
    --quiet

gcloud compute networks delete $NETWORK \
    --quiet
```