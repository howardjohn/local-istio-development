# local-istio-development
Scripts and documentation for developing Istio locally

## Fully Cloud

This setup involves a fully cloud developement setup, with a remote docker registry.

At a basic level, the entire Istio codebase can be built and pushed, then Istio can be installed to point to our custom images.

TIP: using the `dockerx` targets uses BuildKit which is a bit faster than `docker.push`.

```shell
export HUB=my-docker-hub
export TAG=my-build
make dockerx.pushx
istioctl install --set hub=${HUB} --set tag=${TAG}
```

However, this is very slow, as it builds a lot of docker images and requires a full rollout.

For rapid iteration on a single component, like Istiod, its better to just build and push a single image, then replace it inline:

```shell
istiod-build() {
        BUILD_ALL=false DOCKER_TARGETS=docker.pilot make dockerx.pushx \
        && kubectl -n istio-system set image deployment/istiod discovery=${HUB}/pilot:${TAG} --record
}
HUB=my-docker-hub TAG=my-build istiod-build
```

## Local Cluster + Registry

To avoid heavy network traffic pushing and pulling images, a local registry can be used.

For a local registry, docker offers the `registry` image.

For local Kubernetes clusters, there are many options such as [kind](https://kind.sigs.k8s.io/), [minikube](https://minikube.sigs.k8s.io/docs/), [k3d](https://k3d.io/), and more. I strongly recommend `kind`.

An example end to end setup with kind can be found in the [kind documentation](https://kind.sigs.k8s.io/docs/user/local-registry/#create-a-cluster-and-registry).

With this, we can follow the sames steps as before for development, but replace the `HUB` with the local registry, such as `HUB=localhost:5000`.

One other benefit is the local Kubernetes cluster will (likely) not have any rate limitting allowing things to run faster.

## Fully Local

Both the proxy and Istiod can be run locally. The proxy likely requires linux, however.

Istiod can be run as simply `go run ./pilot/cmd/pilot-discovery discovery`. This will connect to the locally configured Kubernetes cluster, configured by the kubectl context, where all configuration can be read.

The proxy can similarly be run, although it does require some configuration:

I typically setup a `proxy-config.yaml`:
```yaml
binaryPath: $GOPATH/src/istio.io/istio/out/linux_amd64/envoy
configPath: $HOME/kube/local/proxy
proxyBootstrapTemplatePath: $GOPATH/src/istio.io/istio/tools/packaging/common/envoy_bootstrap.json
discoveryAddress: localhost:15012
statusPort: 15020
terminationDrainDuration: 0s
tracing: {}
```

Then fetch the root certificate and a token:

```shell
proxy-local-bootstrap() {
    mkdir -p ./var/run/secrets/tokens ./var/run/secrets/istio
    echo '{"kind":"TokenRequest","apiVersion":"authentication.k8s.io/v1","spec":{"audiences":["istio-ca"], "expirationSeconds":2592000}}' | \
        kubectl create --raw /api/v1/namespaces/${1:-default}/serviceaccounts/${2:-default}/token -f - | jq -j '.status.token' > ./var/run/secrets/tokens/istio-token
    kubectl -n istio-system get secret istio-ca-secret -ojsonpath='{.data.ca-cert\.pem}' | base64 -d > ./var/run/secrets/istio/root-cert.pem
}
proxy-local-bootstrap
```

This can then be run with:
```shell
PROXY_CONFIG="$(< ~/kube/local/proxyconfig.yaml envsubst)" go run ./pilot/cmd/pilot-agent proxy sidecar
```

This will run a proxy for an arbitrary connection in the `default` namespace. This may not exactly mirror a real proxy, which is associated with a pod which may impact the configuration generated.

We can "clone" a running pod by impersonating it:

```shell
proxy-clone() {
    labels="$(kubectl get pods "${1:?pod name}" -ojsonpath='{.metadata.labels}')"
    namespace="$(kubectl get pods "${1:?pod name}" -ojsonpath='{.metadata.namespace}')"
    sa="$(kubectl get pods "${1:?pod name}" -ojsonpath='{.spec.serviceAccountName}')"
    proxy-local-bootstrap ${namespace} ${sa}
    ISTIO_META_NAMESPACE="$namespace" ISTIO_META_CLUSTER_ID=Kubernetes ISTIO_METAJSON_LABELS="$labels" PROXY_CONFIG="$(< ~/kube/local/proxyconfig.yaml envsubst)" go run ./pilot/cmd/pilot-agent proxy sidecar
}
proxy-clone hello-world-69898fd668-46wxp
```

We can "clone" a gateway locally as well:

```shell
gateway-local() {
    proxy-local-bootstrap istio-system istio-ingressgateway-service-account
    ISTIO_META_NAMESPACE="istio-system" ISTIO_META_CLUSTER_ID=Kubernetes ISTIO_METAJSON_LABELS='{"istio": "ingressgateway", "app": "istio-ingressgateway"}' PROXY_CONFIG="$(< ~/kube/local/proxyconfig.yaml envsubst)" go run ./pilot/cmd/pilot-agent proxy router
}
gateway-local
```
