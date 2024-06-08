# k8s-devsecops

## How is this repo organized?
This repository builds on top of the directory structure mentioned [here](https://github.com/fluxcd/flux2-kustomize-helm-example/tree/main?tab=readme-ov-file#repository-structure) except that it is collapsed into a top-level directory called `./kubernetes` and the `infrastructure` directory is renamed to `components`.

## Setup

You'll need the following tools:
- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/)
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

Once all pods are ready, configure a port-forward for the HTTPS endpoint of Istio gatewaywith:

```sh
kubectl port-forward -n istio-ingress svc/istio-ingressgateway 8443:443
```

Run the following command to ping the gateway:

```sh
export SECURE_INGRESS_PORT=8443
export INGRESS_HOST=127.0.0.1
curl -v -k -HHost:apps.example.com --resolve "apps.example.com:${SECURE_INGRESS_PORT}:$INGRESS_HOST" "https://apps.example.com:${SECURE_INGRESS_PORT}/"
```

## Characteristics

- Every kubernetes manifest in this repository is [continously reconciled](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/clusters/kind/flux-system/gotk-sync.yaml) via Flux.
- Uses a default deny all `AuthorizationPolicy` resource to deny all L7 communications between pods. Each traffic flow must be explicitly allowed by defining an `AuthorizationPolicy` resource. See [this](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/apps/kind/allow-ingress-to-nginx.yaml) for example.
- Uses mesh-wide strict mTLS using [`PeerAuthentication` resource](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/strict-mtls.yaml), therefore, every pod needs to have a certificate issued by the Istio CA to talk to another pod within the mesh. This in combination with an `AuthorizationPolicy` adds [service-to-service authentication](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/apps/kind/allow-ingress-to-nginx.yaml).
- Every request passes through the ingress gateway and automatically [redirects HTTP to HTTPS](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/gateway.yaml#L16-L17).
- A [`WasmPlugin` resource](https://github.com/vedantthapa/k8s-devsecops/blob/main/kubernetes/components/configs/waf.yaml) is used for configuring WAFs on the Istio ingress gateway. A similar resource can be defined for individual pods within the mesh.

## Acknowledgements
[Istio Best Practices](https://istio.io/latest/docs/ops/best-practices/security/)

[flux2-kustomize-helm-example](https://github.com/fluxcd/flux2-kustomize-helm-example)

[Istio Day - (Almost) Secure by Default - Next Steps for Hardening Istio in Production Environments](https://www.youtube.com/watch?v=C4hADTuyGYc)

[Cloud-native SecurityCon - CNI or Service Mesh? Comparing Security Policies Across Providers](https://www.youtube.com/watch?v=L5UifNZCKhA&t)

[Istio Day - Lightning Talk: Use Wasm to Deploy WAF Deeper in the Service Mesh for Zero Trust and Compliance](https://www.youtube.com/watch?v=FPxAvjghJ3E)

[Netpol with Istio](https://istio.io/v1.10/blog/2017/0.1-using-network-policy/)

[Could network cache based identity be mistaken?](https://www.solo.io/blog/could-network-cache-based-identity-be-mistaken/)
