# local-istio-development

Scripts and documentation for developing Istio locally. This contains supplemental information to my talk at [IstioCon 2021: Local Istio Development](https://events.istio.io/istiocon-2021/sessions/local-istio-development/).

For the slides from the talk, see [slides.pdf](slides.pdf).

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

If you use `kind` and are getting `ResoureNotFound` errors, your cluster might need to be [set up](https://kind.sigs.k8s.io/docs/user/configuration/#getting-started) with a `--config` flag pointing to [`prow/config/trustworthy-jwt.yaml`](https://github.com/istio/istio/blob/master/prow/config/trustworthy-jwt.yaml). 

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

TIP: go can utilize caches better when not using `go run`, and when compiling multiple binaries at once. This can be utilized to reduce the time to run:
```shell
alias build-local='go build -o ./out/linux_amd64 ./pilot/cmd/pilot-agent ./pilot/cmd/pilot-discovery'
alias istiod-local='build-local && ./out/linux_amd64/pilot-discovery discovery
```

## Remote Istiod, local proxy

Connecting a local proxy to a remote Istiod can be done pretty simply by using `port-forward`:

```shell
kubectl port-forward -n istio-system svc/istiod 15012
```

Then running the same commands as above to run the proxy locally.

## Local Istiod, remote proxy

This may require following [Enable forwarding from Docker containers to the outside world(https://docs.docker.com/network/bridge/#enable-forwarding-from-docker-containers-to-the-outside-world)first.

To have proxies running the cluster connect to our local Istio, we can replace the `Endpoints` for the `istiod` Service to point to our locally running instance.

```shell
use-local-pilot() {
  ip=$(/sbin/ip route | awk '/docker0/ { print $9 }')
  kubectl -n istio-system delete svc istiod
  kubectl -n istio-system delete endpoints istiod
  cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: istiod
  namespace: istio-system
spec:
  ports:
  - name: grpc-xds
    port: 15010
  - name: https-dns
    port: 15012
  - name: https-webhook
    port: 443
    targetPort: 15017
  - name: http-monitoring
    port: 15014
---
apiVersion: v1
kind: Endpoints
metadata:
  name: istiod
  namespace: istio-system
subsets:
- addresses:
  - ip: ${ip}
  ports:
  - name: https-dns
    port: 15012
    protocol: TCP
  - name: grpc-xds
    port: 15010
    protocol: TCP
  - name: https-webhook
    port: 15017
    protocol: TCP
  - name: http-monitoring
    port: 15014
    protocol: TCP

EOF
}
```

This may require restarting pods to have them re-establish the connection.

## Plain Envoy

To avoid dependency on Istio entirely, we can run Envoy directly with a static configuration. For example, a barebones configuration with no XDS configuration setup:
```yaml
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 10000
```

This can then be run with `envoy -c envoy.yaml`.

From the `istio/istio` repository, the `envoy` binary will be stored in `out/linux_amd64/release/envoy` after running `make init`.

See [Envoy examples](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/examples) for more information on configuring Envoy manually.

## Direct clients

We can also connect directly to Istiod through various clients.

One useful one is [grpcurl](https://github.com/fullstorydev/grpcurl).

```shell
toJson () {
        python -c '
import sys, yaml, json
yml = list(y for y in yaml.safe_load_all(sys.stdin) if y)
if len(yml) == 1: yml = yml[0]
json.dump(yml, sys.stdout, indent=4)
'
}

token=$(echo '{"kind":"TokenRequest","apiVersion":"authentication.k8s.io/v1","spec":{"audiences":["istio-ca"], "expirationSeconds":2592000}}' | kubectl create --raw /api/v1/namespaces/istio-system/serviceaccounts/istio-ingressgateway-service-account/token -f - | jq -j '.status.token')
request=$(cat request.yaml | toJson)
echo "${request}" | grpcurl -d @ -insecure -rpc-header "authorization: Bearer $token" localhost:15012 envoy.service.discovery.v3.AggregatedDiscoveryService/StreamAggregatedResources
```

Where `request.yaml` is a `DiscoveryRequest` object, for example:
```yaml
node:
  id: router~10.244.0.36~foo.istio-system~istio-system.svc.cluster.local
  metadata:
    CONFIG_NAMESPACE: istio-system
typeUrl: type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret
resourceNames:
- kubernetes://sds-credential
```

[pilot-load](https://github.com/howardjohn/pilot-load) is another similar tool that is specialized for Istiod.
