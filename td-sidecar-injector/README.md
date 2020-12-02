# Istio sidecar proxy injector installation for GKE

To install Istio sidecar proxy injector on GKE to be used with Traffic Director
two steps are necessary:

1. Create the secret for the sidecar injector:
   * key.pem --- the private key for the sidecar injector workload.
   * cert.pem --- the certificate of the sidecar injector workload.
   * ca-cert.pem --- the certificate of the signing CA.

```
kubectl apply -f specs/00-namespaces.yaml

kubectl create secret generic istio-sidecar-injector -n istio-control \
  --from-file=key.pem \
  --from-file=cert.pem \
  --from-file=ca-cert.pem
```

2.  Edit `specs/01-configmap.yaml` file:
    * replace `my-project-here` with your project number
    * replace `your-network-here` with the network name

3.  Deploy Istio sidecar injector by running:

    ```
    kubectl apply -f specs/
    ```

4.  Enable sidecar injection for the namespace by running following command,
    replacing <ns> with the name of the namespace:

    ```
    kubectl label namespace <ns> istio-injection=enabled
    ```

The following annotations are supported:
* sidecar.istio.io/inject
  * This is a Pod annotation. It specifies whether or not an Envoy sidecar
    should be automatically injected into the workload.
* cloud.google.com/proxyMetadata
  * This is a Pod annotation. The key/value pairs in this JSON map will be
    appended to Envoy metadata.
