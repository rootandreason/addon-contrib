# istio-mesh-addon

istio-mesh-addon is an OCM addon that sets up an [Istio multi-primary, multi-network service mesh](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)
across managed clusters using the [AddOnTemplate](https://open-cluster-management.io/concepts/addon/#addontemplate)
framework. The Istio control plane on each cluster is installed by the maintained
[Sail Operator](https://github.com/istio-ecosystem/sail-operator).

## Why this addon?

A multi-primary mesh needs the same Istio configuration on every member cluster, with only the cluster name and
network changing between them. Without this addon, a platform team has to create the `Istio` CR and the east-west
gateway on each cluster by hand and keep their versions in sync.

This addon delivers that configuration from the hub, once:

- A cluster joins the mesh when it gets the `addon.open-cluster-management.io/istio-mesh=enabled` label
- `{{CLUSTER_NAME}}` fills in the Istio cluster name and network per cluster
- The Istio version and mesh ID are changed in one `AddOnDeploymentConfig` on the hub

It replaces the archived [multicluster-mesh](https://github.com/open-cluster-management-io/multicluster-mesh) addon,
which installed Istio 1.16 through the in-cluster IstioOperator that Istio removed in 1.24.

## What Gets Deployed

On every cluster selected by the `istio-mesh` Placement, the addon creates (namespaced objects in `istio-system`):

| Object | Purpose |
|---|---|
| `ClusterRole` `open-cluster-management:istio-mesh:work` | lets the OCM work agent manage the `Istio` CR and Istio `Gateway`s |
| `Istio` `default` (`sailoperator.io/v1`) | Istio control plane, with `meshID`, `clusterName` and `network: network-<cluster>` |
| `Deployment`, `ServiceAccount`, `PodDisruptionBudget`, `HorizontalPodAutoscaler` (2-5 replicas) `istio-eastwestgateway` | east-west gateway workload |
| `Gateway` `cross-network-gateway` | exposes `*.local` services on port 15443 to the other networks |

## Prerequisites

You need an OCM hub with managed clusters, for example from the
[setup dev environment](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/setup-dev-environment)
(`kind-hub`, `kind-cluster1`, `kind-cluster2`). The examples below use those contexts.

Each managed cluster needs at least two schedulable nodes: the east-west gateway runs two or more replicas on
separate nodes. On kind, create the managed clusters with two worker nodes, for example:

```bash
cat <<EOF | kind create cluster --name cluster1 --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF
```

Tools: `kubectl`, [helm](https://helm.sh), [istioctl](https://istio.io/latest/docs/setup/getting-started/#download)
matching the Istio version and `openssl`.

On kind, install [cloud-provider-kind](https://github.com/kubernetes-sigs/cloud-provider-kind) and keep it running,
so the east-west gateways get a LoadBalancer address (on macOS it must run with `sudo`):

```bash
sudo cloud-provider-kind
```

Run the following steps on **each managed cluster** before enabling the addon.

### 1. Install the Sail Operator

```bash
helm repo add sail-operator https://istio-ecosystem.github.io/sail-operator
helm repo update sail-operator
for c in cluster1 cluster2; do
  helm install sail-operator sail-operator/sail-operator --version 1.31.1 \
    -n sail-operator --create-namespace --kube-context kind-${c}
done
```

### 2. Create `istio-system` with its network label and the `cacerts` secret

All clusters must share a root CA so that mTLS works across clusters. The `cacerts` secret has to exist **before**
istiod starts; otherwise istiod generates a self-signed CA and cross-cluster traffic fails.

Generate a root CA and an intermediate CA per cluster with the `tools/certs` Makefile shipped in the Istio release
directory. Download the release (it also contains `istioctl`) and point `ISTIO_DIR` at it:

```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.31.1 sh -
export ISTIO_DIR=${PWD}/istio-1.31.1
export PATH=${ISTIO_DIR}/bin:${PATH}
```

```bash
mkdir -p certs && cd certs
make -f ${ISTIO_DIR}/tools/certs/Makefile.selfsigned.mk root-ca
for c in cluster1 cluster2; do
  make -f ${ISTIO_DIR}/tools/certs/Makefile.selfsigned.mk ${c}-cacerts
  kubectl --context kind-${c} create namespace istio-system
  kubectl --context kind-${c} label namespace istio-system topology.istio.io/network=network-${c}
  kubectl --context kind-${c} -n istio-system create secret generic cacerts \
    --from-file=${c}/ca-cert.pem --from-file=${c}/ca-key.pem \
    --from-file=${c}/root-cert.pem --from-file=${c}/cert-chain.pem
done
cd ..
```

> These self-signed CAs are for demos only. Use [cert-manager with istio-csr](https://cert-manager.io/docs/usage/istio-csr/)
> or an external CA in production.

### 3. Create the east-west gateway Service

istiod finds the gateway of a network by the `topology.istio.io/network` label on its Service. An AddOnTemplate
cannot template `metadata.labels`, so this Service is applied per cluster. Its network label and selector must match
the gateway `Deployment` in `deploy/addontemplate.yaml` (`network-<cluster>`):

```bash
for c in cluster1 cluster2; do
  kubectl --context kind-${c} apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: istio-eastwestgateway
  namespace: istio-system
  labels:
    app: istio-eastwestgateway
    istio: eastwestgateway
    topology.istio.io/network: network-${c}
spec:
  type: LoadBalancer
  selector:
    app: istio-eastwestgateway
    istio: eastwestgateway
    topology.istio.io/network: network-${c}
  ports:
  - name: status-port
    port: 15021
    targetPort: 15021
  - name: tls
    port: 15443
    targetPort: 15443
  - name: tls-istiod
    port: 15012
    targetPort: 15012
  - name: tls-webhook
    port: 15017
    targetPort: 15017
EOF
done
```

The same Service is also kept as a manifest in the OCM
[run-multicluster-servicemesh](https://github.com/open-cluster-management-io/ocm/tree/main/solutions/run-multicluster-servicemesh)
solution.

## Quick Start

### 1. Install the addon on the Hub cluster

**Option A: Direct from GitHub (no clone needed)**

```bash
kubectl --context kind-hub apply -k https://github.com/open-cluster-management-io/addon-contrib/istio-mesh-addon/deploy
```

**Option B: Clone and apply**

```bash
git clone https://github.com/open-cluster-management-io/addon-contrib.git
cd addon-contrib/istio-mesh-addon
kubectl --context kind-hub apply -k deploy/
```

### 2. Enable on the managed clusters

```bash
for c in cluster1 cluster2; do
  kubectl --context kind-hub label managedcluster ${c} addon.open-cluster-management.io/istio-mesh=enabled
done
```

The addon-manager deploys the `Istio` CR and the east-west gateway to the labelled clusters.

### 3. Check the deployment

```bash
$ kubectl --context kind-hub get managedclusteraddon -A | grep istio-mesh
cluster1   istio-mesh   True        False
cluster2   istio-mesh   True        False

$ kubectl --context kind-cluster1 wait istio/default --for=condition=Ready --timeout=5m
$ kubectl --context kind-cluster1 -n istio-system get svc istio-eastwestgateway
```

The east-west gateway Service must get an external IP before you continue.

### 4. Connect the clusters

Each cluster needs a remote secret for every other cluster so istiod can discover its endpoints. On kind, the API
server address must be reachable from the other cluster, so use the control-plane node IP:

```bash
for target in cluster1 cluster2; do
  for src in cluster1 cluster2; do
    [ "${src}" = "${target}" ] && continue
    ip=$(kubectl --context kind-${src} get nodes -l node-role.kubernetes.io/control-plane \
      -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
    istioctl create-remote-secret --context kind-${src} --name ${src} --server https://${ip}:6443 \
      | kubectl --context kind-${target} apply -f -
  done
done
```

## Verify cross-cluster traffic

Deploy `helloworld` v1 to `cluster1`, v2 to `cluster2`, and the `sleep` client to both:

```bash
SAMPLES=https://raw.githubusercontent.com/istio/istio/1.31.1/samples
deploy() {  # <cluster> <version>
  kubectl --context kind-$1 create namespace sample
  kubectl --context kind-$1 label namespace sample istio-injection=enabled
  kubectl --context kind-$1 apply -n sample -f ${SAMPLES}/helloworld/helloworld.yaml -l service=helloworld
  kubectl --context kind-$1 apply -n sample -f ${SAMPLES}/helloworld/helloworld.yaml -l version=$2
  kubectl --context kind-$1 apply -n sample -f ${SAMPLES}/sleep/sleep.yaml
}
deploy cluster1 v1
deploy cluster2 v2
```

Send 20 requests from each cluster:

```bash
for c in cluster1 cluster2; do
  echo "== 20 requests from ${c}"
  for i in $(seq 1 20); do
    kubectl --context kind-${c} -n sample exec deploy/sleep -c sleep -- \
      curl -sS --max-time 5 helloworld.sample:5000/hello
  done | sort | uniq -c
done
```

Each cluster gets answers from both versions. Since v2 only exists on `cluster2`, v2 answers on `cluster1` prove that
traffic crosses clusters through the east-west gateways:

```
== 20 requests from cluster1
   5 Hello version: v1, instance: helloworld-v1-79564d84f6-lttvx
  15 Hello version: v2, instance: helloworld-v2-59c8fc8d6d-flkdt
== 20 requests from cluster2
   8 Hello version: v1, instance: helloworld-v1-79564d84f6-lttvx
  12 Hello version: v2, instance: helloworld-v2-59c8fc8d6d-flkdt
```

## Files

```
istio-mesh-addon/
├── README.md
└── deploy/
    ├── kustomization.yaml            # Enables kubectl apply -k
    ├── placement.yaml                # Namespace, ManagedClusterSetBinding, Placement (label-based)
    ├── addondeploymentconfig.yaml    # ISTIO_VERSION, MESH_ID, install namespace
    ├── addontemplate.yaml            # Work agent RBAC, Istio CR, east-west gateway, cross-network Gateway
    └── clustermanagementaddon.yaml   # Registers the addon with the addon-manager
```

## Configuration

The addon is configured by the `istio-mesh-config` `AddOnDeploymentConfig`:

| Variable | Default | Description |
|---|---|---|
| `ISTIO_VERSION` | `v1.31.1` | Istio version installed by the Sail Operator; must be supported by the installed Sail Operator |
| `MESH_ID` | `mesh1` | Istio mesh ID shared by all member clusters |

To upgrade Istio, upgrade the Sail Operator on the managed clusters first, then change `ISTIO_VERSION` and re-apply:

```bash
kubectl --context kind-hub apply -k deploy/
```

## Notes

- The `AddOnDeploymentConfig` sets `agentInstallNamespace: istio-system`. If it is left out, the CRD defaults it to
  `open-cluster-management-agent-addon` and the template agent moves every namespaced object of the `AddOnTemplate`
  into that namespace.
- The OCM work agent is bound to the `admin` ClusterRole, which does not cover the `Istio` CR or Istio's `Gateway`.
  The template's first manifest, the `open-cluster-management:istio-mesh:work` ClusterRole, adds them through the
  `open-cluster-management.io/aggregate-to-work` label. On the first apply the `Istio` CR may be rejected until the
  aggregation lands; the work agent retries.
- The Placement selects from the `global` ManagedClusterSet, which OCM creates and maintains only while the
  `DefaultClusterSet` feature gate is enabled (the default). If you disable the gate, edit `deploy/placement.yaml` to
  bind and select a ManagedClusterSet that contains the mesh clusters instead.
- The east-west gateway runs at least two replicas, each on a separate node, so its `PodDisruptionBudget`
  (`minAvailable: 1`) does not block node drains. It has no access to Kubernetes Secrets: the `AUTO_PASSTHROUGH`
  server does not terminate TLS.
- An `AddOnTemplate` replaces every `{{...}}` it contains. That is why only the `Istio` CR and the gateway are
  templated, and istiod is rendered on the cluster by the Sail Operator.

## Troubleshooting

| Symptom | Cause | Solution |
|---|---|---|
| Gateway pod in `ImagePullBackOff` with image `auto` | The gateway pod was created before istiod's injection webhook was ready | `kubectl -n istio-system rollout restart deploy/istio-eastwestgateway` |
| An `istio-eastwestgateway` pod stays `Pending` | Fewer schedulable nodes than gateway replicas; the replicas must run on separate nodes | Add a node to the cluster (on kind, recreate it with two workers) |
| Addon not `Available`, ManifestWork keeps reporting forbidden on `istios` | The `istio-mesh:work` ClusterRole was not aggregated | Check `kubectl get clusterrole open-cluster-management:istio-mesh:work` on the managed cluster and the work agent's aggregated ClusterRole |
| Placement selects no clusters | `global` ManagedClusterSet missing (`DefaultClusterSet` gate disabled) | Use a ManagedClusterSet that contains the mesh clusters in `deploy/placement.yaml` |
| East-west gateway Service has no external IP | No LoadBalancer implementation | On kind, run `cloud-provider-kind` |
| Requests only reach the local version | Remote secrets missing, or clusters do not share a root CA | Re-run "Connect the clusters"; check that `cacerts` existed before istiod started |

```bash
kubectl --context kind-hub get managedclusteraddon istio-mesh -n cluster1 -o yaml
kubectl --context kind-hub get manifestwork -n cluster1 | grep istio-mesh
```

## Clean up

```bash
for c in cluster1 cluster2; do kubectl --context kind-${c} delete namespace sample --ignore-not-found; done
kubectl --context kind-hub delete -k deploy/
for c in cluster1 cluster2; do
  kubectl --context kind-hub label managedcluster ${c} addon.open-cluster-management.io/istio-mesh-
  # the Istio CR must be gone before the operator is removed, so its finalizer can run
  kubectl --context kind-${c} wait --for=delete istio/default --timeout=2m
  kubectl --context kind-${c} delete namespace istio-system
  helm uninstall sail-operator -n sail-operator --kube-context kind-${c}
  kubectl --context kind-${c} delete clusterrole open-cluster-management:istio-mesh:work --ignore-not-found
done
```

## Related Links

- [Sail Operator](https://github.com/istio-ecosystem/sail-operator)
- [Istio multi-primary on different networks](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)
- [OCM AddOnTemplate documentation](https://open-cluster-management.io/concepts/addon/#addontemplate)
