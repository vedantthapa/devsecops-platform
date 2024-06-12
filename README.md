# k8s-devsecops

## How is this repo organized?

This repository builds on top of the directory structure mentioned [here](https://github.com/fluxcd/flux2-kustomize-helm-example/tree/main?tab=readme-ov-file#repository-structure) with some minor differences -

- The main directories from the reference (`./apps`, `./infrastructure` and `./clusters`) are collapsed into a top-level directory called `./kubernetes`
- The `infrastructure` directory is renamed to `components`.

### How are applications organized?

```sh
./kubernetes/apps
├── base
│   ├── httpbin
│   │   ├── httpbin.yaml
│   │   └── kustomization.yaml
│   └── nginx
│       ├── kustomization.yaml
│       └── nginx.yaml
├── gke
└── kind
    ├── httpbin
    │   ├── kustomization.yaml
    │   ├── ns.yaml
    │   └── vs.yaml
    └── nginx
        ├── allow-ingress-to-nginx.yaml
        ├── deployment-patch.yaml
        ├── kustomization.yaml
        ├── ns.yaml
        └── vs.yaml

8 directories, 12 files
```

Applications manifests in the `./kubernetes/apps` directory are organized as [kustomize bases and overlays](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays). In this case, the overlays are environment specific i.e, `kind` and `gke`. The idea is to keep the application manifests platform agnostic in the `base` directory and store any environment specific resource or patch in it's own overlay directory at either `./kubernetes/apps/kind` or `./kubernetes/apps/gke`. The manifests are further segregated at a namespace-level within their respective base and overlay directories. For instance, all manifests related to the `httpbin` namespace for a kind cluster are located in the `./kubernetes/apps/base/httpbin` and `./kubernetes/apps/kind/httpbin` directories. This allows us to specify dependencies between different applications whenever it's necessary.

## Setup

You'll need the following tools:

- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [istioctl](https://cloud.google.com/service-mesh/v1.20/docs/downloading-istioctl)
- [Taskfile](https://taskfile.dev/)
- [Flux CLI](https://fluxcd.io/flux/cmd/)

### kind

Start by creating a [kind](https://kind.sigs.k8s.io/docs/user/quick-start#installation) cluster with:

```sh
task kind:infra-up
```

Now run the `flux bootstrap git` command with:

```sh
task install-flux
```

This will stand up the entire application on the previously created kind cluster.

Once all pods are ready, configure a port-forward for the HTTPS and HTTP endpoint of Istio gateway with:

```sh
kubectl port-forward -n istio-ingress svc/istio-ingressgateway 8443:443
kubectl port-forward -n istio-ingress svc/istio-ingressgateway 8080:80
```

Run the following command to ping the HTTPS gateway:

```sh
export SECURE_INGRESS_PORT=8443
export INGRESS_HOST=127.0.0.1
curl -v -k -HHost:nginx.kind.com --resolve "nginx.kind.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "https://nginx.kind.com:${SECURE_INGRESS_PORT}/"
```

To ping the HTTP gateway use:

```sh
export SECURE_INGRESS_PORT=8080
export INGRESS_HOST=127.0.0.1
curl -v -k -HHost:nginx.kind.com --resolve "nginx.kind.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "http://nginx.kind.com:${SECURE_INGRESS_PORT}/"
```

> Note the change in protocol and the `SECURE_INGRESS_PORT` variable

## Characteristics

- Every kubernetes manifest in this repository is [continously reconciled](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/clusters/kind/flux-system/gotk-sync.yaml) via Flux.
- [TLS certificates](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/certificate.yaml) are automatically managed via cert-manager.
- Every request passes through the ingress gateway and automatically [redirects HTTP to HTTPS](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/gateway.yaml#L16-L17).

  To verify this on the `kind` cluster created previously, ping the HTTP gateway. You should receive the following response:

  ```sh
  > curl -I -k -HHost:nginx.kind.com --resolve "nginx.kind.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "http://nginx.kind.com:${SECURE_INGRESS_PORT}/"
  HTTP/1.1 301 Moved Permanently
  date: Mon, 10 Jun 2024 17:32:03 GMT
  server: istio-envoy
  transfer-encoding: chunked
  ```

- A [`WasmPlugin` resource](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/waf.yaml) is used for configuring WAFs on the Istio ingress gateway. A similar resource can be defined for individual pods within the mesh.

  To verify this on the `kind` cluster created previously, simulate an XSS attack with:

  ```sh
  > curl -I -k -HHost:nginx.kind.com --resolve "nginx.kind.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "https://nginx.kind.com:${SECURE_INGRESS_PORT}/?arg=<script>alert(0)</script>"
  HTTP/1.1 403 Forbidden
  date: Mon, 10 Jun 2024 17:52:04 GMT
  server: istio-envoy
  transfer-encoding: chunked
  ```

- Uses a default deny all `AuthorizationPolicy` resource to deny all L7 communications between pods in the mesh. Traffic flow must be explicitly allowed by defining an `AuthorizationPolicy` resource. See [this](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/apps/kind/nginx/allow-ingress-to-nginx.yaml) for example.

  To verify this on the `kind` cluster created previously, ping the httpbin service:

  ```sh
  > curl -I -k -HHost:httpbin.kind.com --resolve "httpbin.kind.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "https://httpbin.kind.com:${SECURE_INGRESS_PORT}/"
  HTTP/1.1 403 Forbidden
  content-length: 19
  content-type: text/plain
  date: Mon, 10 Jun 2024 17:58:04 GMT
  server: istio-envoy
  x-envoy-upstream-service-time: 6
  ```

  It returns a `403` because no explicit `AuthorizationPolicy` is set to allow traffic from the ingress gateway to `httpbin` service.

- Uses mesh-wide strict mTLS using [`PeerAuthentication` resource](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/strict-mtls.yaml), therefore, every pod needs to have a certificate issued by the Istio CA to talk to another pod within the mesh. This in combination with an `AuthorizationPolicy` adds [service-to-service authentication](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/apps/kind/nginx/allow-ingress-to-nginx.yaml#L11-L15).

- Optionally, a combination of `RequestAuthentication` + `AuthorizationPolicy` resource can be set up to [only allow requests that contain a JWT token](https://github.com/vedantthapa/istio-oauth2/blob/main/istio/authnz/ingress-jwt.yaml). To take this idea a step further, [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy) can be used to obtain a JWT token from the cloud provider.

## Acknowledgements

[Istio Best Practices](https://istio.io/latest/docs/ops/best-practices/security/)

[flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example)

[Istio Day - (Almost) Secure by Default - Next Steps for Hardening Istio in Production Environments](https://www.youtube.com/watch?v=C4hADTuyGYc)

[Cloud-native SecurityCon - CNI or Service Mesh? Comparing Security Policies Across Providers](https://www.youtube.com/watch?v=L5UifNZCKhA&t)

[Istio Day - Lightning Talk: Use Wasm to Deploy WAF Deeper in the Service Mesh for Zero Trust and Compliance](https://www.youtube.com/watch?v=FPxAvjghJ3E)

[Netpol with Istio](https://istio.io/v1.10/blog/2017/0.1-using-network-policy/)

[Could network cache based identity be mistaken?](https://www.solo.io/blog/could-network-cache-based-identity-be-mistaken/)

[zta-system-patterns](https://github.com/PHACDataHub/zta-system-pattern)

[node-microservices-demo](https://github.com/PHACDataHub/node-microservices-demo)
