# Traffic Director Demo
In this example, we demostrate how services in two cross-regions GKE clusters could communite with each other through Traffic Director. Traffic Director can do service discovery cross clusters and dynamically configure service proxy for load balancing and fail over control.

## Reference
* https://cloud.google.com/load-balancing/docs
* https://cloud.google.com/vpc/docs/firewalls
* https://cloud.google.com/traffic-director/docs/setting-up-traffic-director
* https://cloud.google.com/traffic-director/docs/set-up-gke-pods-auto
* https://cloud.google.com/traffic-director/docs/configure-advanced-traffic-management
* https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority#arch-overview-load-balancing-priority-levels

## GKE Pods with automatic Envoy injection
1. Create GKE clusters

```bash
# GKE in us-central1
$ gcloud container clusters create gke-us-central1 \
    --zone us-central1-a \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --enable-ip-alias
# Context
$ gcloud container clusters get-credentials gke-us-central1 \
    --zone us-central1-a

# GKE in asia-east1
$ gcloud container clusters create gke-asia-east1 \
    --zone asia-east1-a \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --enable-ip-alias
# Context
$ gcloud container clusters get-credentials gke-asia-east1 \
    --zone asia-east1-a
```

1. Install sidecar injector

* Project config
```bash
$ wget https://storage.googleapis.com/traffic-director/td-sidecar-injector.tgz && \
  tar -xzvf td-sidecar-injector.tgz
$ cd td-sidecar-injector
```

Open `specs/01-configmap.yaml`

Update `TRAFFICDIRECTOR_GCP_PROJECT_NUMBER` & `TRAFFICDIRECTOR_NETWORK_NAME`

* Configuring TLS for the sidecar injector
```bash
CN=istio-sidecar-injector.istio-control.svc

openssl req \
  -x509 \
  -newkey rsa:4096 \
  -keyout key.pem \
  -out cert.pem \
  -days 365 \
  -nodes \
  -subj "/CN=${CN}"

cp cert.pem ca-cert.pem
```

* Create kubernetes resources
```bash
$ kubectl apply -f specs/00-namespaces.yaml

$ kubectl create secret generic istio-sidecar-injector -n istio-control \
  --from-file=pem/key.pem \
  --from-file=pem/cert.pem \
  --from-file=pem/ca-cert.pem

$ CA_BUNDLE=$(cat cert.pem | base64 | tr -d '\n')
$ sed -i "s/caBundle:.*/caBundle:\ ${CA_BUNDLE}/g" specs/02-injector.yaml

$ kubectl apply -f specs/
$ kubectl get pods -A | grep sidecar-injector
```

* Enabling sidecar injection
```bash
$ kubectl label namespace default istio-injection=enabled
$ kubectl get namespace -L istio-injection
```

## Deploy Demo Apps/Services

```bash
# Client app
$ kubectl create -f demo/client_sample.yaml
$ kubectl describe pods -l run=client

# Service
$ kubectl create -f demo/service_us.yaml
$ kubectl create -f demo/service_asia.yaml
```

## Configure Load Balancing components

* Cloud Load Balancing components

```bash
# Creating the health check and firewall rule
$ gcloud compute health-checks create http td-gke-health-check \
  --use-serving-port

$ gcloud compute firewall-rules create fw-allow-health-checks \
  --action ALLOW \
  --direction INGRESS \
  --source-ranges 35.191.0.0/16,130.211.0.0/22 \
  --target-tags <targets || all> \
  --rules tcp:80

# Creating the backend service
$ gcloud compute backend-services create td-gke-service \
 --global \
 --health-checks td-gke-health-check \
 --load-balancing-scheme INTERNAL_SELF_MANAGED

$ gcloud compute backend-services add-backend td-gke-service \
 --global \
 --network-endpoint-group ${NEG_NAME} \
 --network-endpoint-group-zone us-central1-a \
 --balancing-mode RATE \
 --max-rate-per-endpoint 5

# Creating the routing rule map
$ gcloud compute url-maps create td-gke-url-map \
   --default-service td-gke-service

$ gcloud compute url-maps add-path-matcher td-gke-url-map \
   --default-service td-gke-service \
   --path-matcher-name td-gke-path-matcher

$ gcloud compute url-maps add-host-rule td-gke-url-map \
   --hosts service-test \
   --path-matcher-name td-gke-path-matcher

$ gcloud compute target-http-proxies create td-gke-proxy \
   --url-map td-gke-url-map

$ gcloud compute forwarding-rules create td-gke-forwarding-rule \
  --global \
  --load-balancing-scheme=INTERNAL_SELF_MANAGED \
  --address=0.0.0.0 \
  --target-http-proxy=td-gke-proxy \
  --ports 80 --network default
```

## Verification
* Busybox
```bash
# Get the name of the pod running Busybox.
$ BUSYBOX_POD=$(kubectl get po -l run=client -o=jsonpath='{.items[0].metadata.name}')

# Command to execute that tests connectivity to the service service-test.
$ TEST_CMD="wget -q --header 'Host: service-hello' -O - 10.0.0.1; echo"

# Execute the test command on the pod.
$ kubectl exec -it $BUSYBOX_POD -c busybox -- /bin/sh -c "$TEST_CMD"
```

* Request backend service inside busybox
```bash
$ kubectl exec -it $BUSYBOX_POD -c busybox sh
$ wget -q --header "Host: service-hello" -O - 10.0.0.1; echo
```

* Verify envoy clusters configed by Traffic Director xDS server
```bash
$ kubectl exec -it $BUSYBOX_POD -c istio-proxy -- /bin/bash -c "curl 127.0.0.1:15000/clusters"

# Example output
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::cx_active::2
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::cx_connect_fail::0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::cx_total::4
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::rq_active::0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::rq_error::0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::rq_success::8
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::rq_timeout::0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::rq_total::8
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::hostname::
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::health_flags::healthy
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::weight::1
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::region::
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::zone::
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::sub_zone::jq:us-central1-c_1283358594105812863_neg
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::canary::false
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::priority::0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::success_rate::-1.0
cloud-internal-istio:cloud_mp_636150303269_5729880038847775802::10.108.0.7:80::local_origin_success_rate::-1.0
```
