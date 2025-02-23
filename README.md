# GKE Gateway APIs Experiments

## Requirements

- GCP Project
- GCP User with enough IAM roles on project (Owner for testing purposes)
- Active domain and access to registrar to add CNAME records for Domain validation

## Setup

### Set env vars

```bash
export PROJECT_ID=<PROJECT_ID>
export FQDN_1=<SUBDOMAIN_1.DOMAIN>
export FQDN_1_NAME=<SUBDOMAIN_1_DOMAIN>
export FQDN_2=<SUBDOMAIN_2.DOMAIN>
export FQDN_2_NAME=<SUBDOMAIN_2_DOMAIN>

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
export CERT_MAP=cert-map
export IP_NAME=lb-ext-ip
export SSL_POLICY=strict-ssl
export CLOUD_ARMOR_POLICY=gke-armor

gcloud config set project $PROJECT_ID
```

### Substitute variables in YAML files

```bash
sed -i "s/CERT_MAP/$CERT_MAP/g"  platform-admin/gateway.yaml
sed -i "s/IP_NAME/$IP_NAME/g"  platform-admin/gateway.yaml
sed -i "s/FQDN_1/$FQDN_1/g"  platform-admin/gateway.yaml
sed -i "s/FQDN_2/$FQDN_2/g"  platform-admin/gateway.yaml
sed -i "s/SSL_POLICY/$SSL_POLICY/g" platform-admin/gateway-policy.yaml
sed -i "s/CLOUD_ARMOR_POLICY/$CLOUD_ARMOR_POLICY/g" platform-admin/backend-policy.yaml
sed -i "s/FQDN_1/$FQDN_1/g"  developer-team-1/httproute.yaml
sed -i "s/FQDN_1/$FQDN_1/g"  developer-team-1/cross-ns-httproute.yaml
sed -i "s/FQDN_2/$FQDN_2/g"  developer-team-2/httproute.yaml
```

### Setup networking

```bash
# Create VPC
gcloud compute networks create $NETWORK \
    --subnet-mode=custom

# Create subnetwork
gcloud compute networks subnets create $SUBNETWORK \
    --range=10.100.0.0/20  \
    --network=$NETWORK \
    --region=$REGION

# Enable private nodes access to Internet
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

```bash
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

```bash
# Enable Certificate Manager APIs
gcloud services enable certificatemanager.googleapis.com

# Create Certificate Map, which will be used by the Gateway
gcloud certificate-manager maps create $CERT_MAP \
    --location global \
    --description "Global cert map"


# Create DNS Authorizations for each tenant domain
gcloud certificate-manager dns-authorizations create $FQDN_1_NAME \
    --domain $FQDN_1 \
    --location global

gcloud certificate-manager dns-authorizations create $FQDN_2_NAME \
    --domain $FQDN_2 \
    --location global

# Create Google-Managed certificate for each tenant domain
gcloud certificate-manager certificates create $FQDN_1_NAME \
    --domains $FQDN_1 \
    --location global \
    --dns-authorizations $FQDN_1_NAME

gcloud certificate-manager certificates create $FQDN_2_NAME \
    --domains $FQDN_2 \
    --location global \
    --dns-authorizations $FQDN_2_NAME

# Add Certificate to corresponding map entry for each tenant domain
gcloud certificate-manager maps entries create $FQDN_1_NAME \
    --location global \
    --map $CERT_MAP \
    --hostname $FQDN_1 \
    --certificates $FQDN_1_NAME

gcloud certificate-manager maps entries create $FQDN_2_NAME \
    --location global \
    --map $CERT_MAP \
    --hostname $FQDN_2 \
    --certificates $FQDN_2_NAME

# Retrieve DNS Authorization Challenge data to validate Certificate with DNS Challenge
CHALLENGE_NAME_FQDN_1=$(gcloud certificate-manager dns-authorizations describe $FQDN_1_NAME --format json | jq -r '.dnsResourceRecord.name')
CHALLENGE_DATA_FQDN_1=$(gcloud certificate-manager dns-authorizations describe $FQDN_1_NAME --format json | jq -r '.dnsResourceRecord.data')
echo $CHALLENGE_NAME_FQDN_1
echo $CHALLENGE_DATA_FQDN_1

CHALLENGE_NAME_FQDN_2=$(gcloud certificate-manager dns-authorizations describe $FQDN_2_NAME --format json | jq -r '.dnsResourceRecord.name')
CHALLENGE_DATA_FQDN_2=$(gcloud certificate-manager dns-authorizations describe $FQDN_2_NAME --format json | jq -r '.dnsResourceRecord.data')
echo $CHALLENGE_NAME_FQDN_2
echo $CHALLENGE_DATA_FQDN_2
```

<b> IMPORTANT: add a CNAME record on your registrar with CHALLENGE_NAME and CHALLENGE_DATA values to validate certificate. Also create the corresponding A record pointing the FQDNs to the LB_IP </b>



### Platform Admin: create namespaces for gateway and application resources

```bash
kubectl apply -f platform-admin/namespaces.yaml
```


### Platform Admin: deploy GKE Gateway controller for Global HTTP/S ALB

```bash
kubectl apply -f platform-admin/gateway.yaml -n gateway
kubectl apply -f platform-admin/https-redirect.yaml -n gateway
```


### Developer Team 1: Deploy applications and httproutes as developer

```bash
kubectl apply -f developer-team-1/app.yaml -n dev-team-1
kubectl apply -f developer-team-1/httproute.yaml -n dev-team-1
```


### Developer Team 1: Test backend applications

Verify response header injection:

```bash
curl -v https://$FQDN_1/
```

Verify canary deployment result with weighted traffic splitting:

```bash
v1=v2=0
for i in {1..100}; do
    [[ $(curl -s https://$FQDN_1) =~ "Dev Team 1 App V1" ]] && ((v1++)) || [[ $(response) =~ "Dev Team 1 App V2" ]] && ((v2++))
    echo -ne "\r$i/100"
done
printf "\nv1: %d (%.1f%%)\nv2: %d (%.1f%%)\n" $v1 $((v1*100/(v1+v2))) $v2 $((v2*100/(v1+v2)))
```

Verify request redirect for specific path:

```bash
curl -L -v https://$FQDN_1/old
```

Verify dark launch pattern:

```bash
curl -H "version: dark" https://$FQDN_1
```

### Developer Team 2: Deploy application and httproutes as developer

```bash
kubectl apply -f developer-team-2/app.yaml -n dev-team-2
kubectl apply -f developer-team-2/httproute.yaml -n dev-team-2
```

Verify application exposure:

```bash
curl -v https://$FQDN_2
```

### Developer Team 2: Cross namespace reference for Dev Team 1

```bash
kubectl apply -f developer-team-2/referencegrant.yaml -n dev-team-2
```

### Developer Team 1: Reference cross-namespace service

```bash
kubectl apply -f developer-team-1/cross-ns-httproute.yaml -n dev-team-1
```

Test access to dev-team-2 backend:

```bash
curl -H "team: dev-team-2" https://$FQDN_1
```

### Developer Team 1: Define custom HealthCheck policies

```bash
kubectl apply -f developer-team-1/hc-policy.yaml -n dev-team-1
```

### Platform Admin: Create strict SSL Policy and apply to Gateway

```bash
gcloud compute ssl-policies create $SSL_POLICY \
    --global \
    --min-tls-version 1.2 \
    --profile RESTRICTED

kubectl apply -f platform-admin/gateway-policy.yaml -n gateway
```

Test HTTP request with unsupported TLS version:

```bash
curl --tls-max 1.1 https://$FQDN_1
curl --tls-max 1.1 https://$FQDN_2
```


### Platform Admin: Create Cloud Armor policy with deny all rule

```bash
gcloud compute security-policies create $CLOUD_ARMOR_POLICY \
    --type CLOUD_ARMOR \
    --global

gcloud compute security-policies rules update 2147483647 \
    --action deny-403 \
    --security-policy $CLOUD_ARMOR_POLICY
```

```bash
kubectl apply -f platform-admin/backend-policy.yaml -n dev-team-1
```

Test HTTP request calling dark backend and verify traffic is blocked:

```bash
curl -H "version: dark" https://$FQDN_1
```

## Cleanup

### Substitute back variables in YAML files

```bash
sed -i "s/$CERT_MAP/CERT_MAP/g"  platform-admin/gateway.yaml
sed -i "s/$IP_NAME/IP_NAME/g"  platform-admin/gateway.yaml
sed -i "s/$FQDN_1/FQDN_1/g"  platform-admin/gateway.yaml
sed -i "s/$FQDN_2/FQDN_2/g"  platform-admin/gateway.yaml
sed -i "s/$S$SL_POLICY/SSL_POLICY/g" platform-admin/gateway-policy.yaml
sed -i "s/$CLOUD_ARMOR_POLICY/CLOUD_ARMOR_POLICY/g"  platform-admin/backend-policy.yaml
sed -i "s/$FQDN_1/FQDN_1/g"  developer-team-1/httproute.yaml
sed -i "s/$FQDN_1/FQDN_1/g"  developer-team-1/cross-ns-httproute.yaml
sed -i "s/$FQDN_2/FQDN_2/g"  developer-team-2/httproute.yaml
```

### Delete GCP resources

```bash
kubectl delete ns dev-team-1 dev-team-2 gateway

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

gcloud compute security-policies delete $CLOUD_ARMOR_POLICY --global --quiet

gcloud compute ssl-policies delete $SSL_POLICY --global --quiet
```