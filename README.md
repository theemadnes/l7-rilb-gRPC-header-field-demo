# l7-rilb-gRPC-header-field-demo
switching from generated cookie to header field using gcloud 

### usage

deploy gRPC app, K8s service and namespace

```
kubectl apply -f whereami-grpc/
```

now let's start creating the load balancer (and related) resources

```
# create a firewall rule to allow health checking against the gRPC service(s) - i'm being lazy here and just allowing against the CIDR range for the gRPC pods across the cluster - you'll want to be more specific 
# replace with your specific port, project, CIDR range, and VPC name, and possibly add the argument to indicate specific tags or service accounts
gcloud compute firewall-rules create grpc-fw-health-check \
    --project e2m-private-test-01 \
    --network=default \
    --action=ALLOW \
    --rules=tcp:9090 \
    --source-ranges=35.191.0.0/16,130.211.0.0/22 \
    --destination-ranges=10.54.0.0/17 \
    --direction=INGRESS \
    --description="Allow gRPC health check probes from Google Cloud Load Balancer ranges."

# create FW rule to allow the L7 proxies to attach to the gRPC service
# replace with the correct project ID, VPC, ranges, etc
gcloud compute firewall-rules create grpc-fw-allow-proxies \
    --network=default \
    --project=e2m-private-test-01 \
    --action=allow \
    --direction=ingress \
    --source-ranges=172.16.10.0/24 \
    --destination-ranges=10.54.0.0/17 \
    --rules=tcp:9090

# create the health check - replace with your region and project ID
gcloud compute health-checks create grpc grpc-header-health-check \
    --region=us-central1 \
    --port=9090 \
    --project e2m-private-test-01

# create backend service
gcloud compute backend-services create grpc-ilb-gke-backend-service \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --protocol=H2C \
    --health-checks=grpc-header-health-check \
    --health-checks-region=us-central1 \
    --region=us-central1 \
    --project e2m-private-test-01

# add NEGs to backend service, making sure to reflect your zones / project / etc
gcloud compute backend-services add-backend grpc-ilb-gke-backend-service \
    --network-endpoint-group=grpc-header \
    --network-endpoint-group-zone=us-central1-a \
    --region=us-central1 \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10 \
    --project e2m-private-test-01

gcloud compute backend-services add-backend grpc-ilb-gke-backend-service \
    --network-endpoint-group=grpc-header \
    --network-endpoint-group-zone=us-central1-b \
    --region=us-central1 \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10 \
    --project e2m-private-test-01

gcloud compute backend-services add-backend grpc-ilb-gke-backend-service \
    --network-endpoint-group=grpc-header \
    --network-endpoint-group-zone=us-central1-c \
    --region=us-central1 \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10 \
    --project e2m-private-test-01

gcloud compute backend-services add-backend grpc-ilb-gke-backend-service \
    --network-endpoint-group=grpc-header \
    --network-endpoint-group-zone=us-central1-f \
    --region=us-central1 \
    --balancing-mode=RATE \
    --max-rate-per-endpoint=10 \
    --project e2m-private-test-01

# create URL map replacing the necessary values
gcloud compute url-maps create grpc-ilb-gke-map \
    --default-service=grpc-ilb-gke-backend-service \
    --region=us-central1 \
    --project e2m-private-test-01

# create target proxy with necessary replacements
gcloud compute target-http-proxies create grpc-ilb-gke-proxy \
    --url-map=grpc-ilb-gke-map \
    --url-map-region=us-central1 \
    --region=us-central1 \
    --project e2m-private-test-01

# create the forwarding rule with necessary replacements
gcloud compute forwarding-rules create grpc-ilb-gke-forwarding-rule \
    --load-balancing-scheme=INTERNAL_MANAGED \
    --network=default \
    --subnet=default \
    --ports=80 \
    --region=us-central1 \
    --target-http-proxy=grpc-ilb-gke-proxy \
    --target-http-proxy-region=us-central1 \
    --project e2m-private-test-01

# capture the VIP
# in my case, it's 10.128.0.81

# update the backend service to implement session affinity based on header_field
gcloud compute backend-services update grpc-ilb-gke-backend-service \
    --project e2m-private-test-01 \
    --region=us-central1 \
    --session-affinity=HEADER_FIELD \
    --custom-request-header="User-Session:" \
    --locality-lb-policy=MAGLEV \
    --consistent-hash-http-header-name="User-Session" 

# dump config of lb
gcloud compute backend-services describe grpc-ilb-gke-backend-service \
    --project=e2m-private-test-01 \
    --region=us-central1

gcloud compute backend-services update grpc-ilb-gke-backend-service \
    --project e2m-private-test-01 \
    --region=us-central1 \
    --locality-lb-policy=MAGLEV

### argh had to just update in GUI to get this to work

```