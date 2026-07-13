# Istio Multi-Cluster GitOps

Deploy Istio across a fleet of OpenShift/Kubernetes clusters using ACM (Advanced Cluster Management) and ArgoCD.

## Prerequisites

- OpenShift or Kubernetes cluster with ACM hub installed
- `kubectl` CLI configured to access the ACM hub cluster
- Managed clusters imported into ACM

## Managed clusters and Placement

### Step 1: Label managed clusters

Label each cluster that should join the Istio mesh. ACM does not copy ManagedCluster labels to the corresponding namespaces on the hub, so both the ManagedCluster and its namespace must be labeled:

```bash
kubectl label managedcluster <cluster-name> mesh=enabled
kubectl label namespace <cluster-name> mesh=enabled
```

Verify the labels:

```bash
kubectl get managedclusters -l mesh=enabled
kubectl get namespaces -l mesh=enabled
```

### Step 2: Create the policy namespace

Create a namespace on the hub cluster to hold ACM policies:

```bash
kubectl create namespace istio-policies
```

### Step 3: Create the Placement for mesh clusters

Apply the Placement and ManagedClusterSetBinding. The ManagedClusterSetBinding grants the `istio-policies` namespace access to the `default` ManagedClusterSet. The Placement selects managed clusters with the label `mesh=enabled` and is shared by all ACM policies targeting mesh clusters (namespaces, certificates, etc.):

```bash
kubectl apply -f acm/service-mesh/placement/
```

## Service Mesh operator

### Step 4: Install the OpenShift Service Mesh 3 operator on managed clusters

Apply the Policy and PlacementBinding that install the Service Mesh operator via OLM on all mesh clusters:

```bash
kubectl apply -f acm/service-mesh/operator/
```

This creates:

- **Policy** (`servicemeshoperator`) — contains a ConfigurationPolicy that enforces a Subscription for the `servicemeshoperator3` package from the `redhat-operators` catalog.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy servicemeshoperator -n istio-policies
```

## Namespace creation

### Step 5: Ensure the istio-system and istio-cni namespaces on managed clusters

Apply the Policy and PlacementBinding that enforce the `istio-system` and `istio-cni` namespaces on all mesh clusters:

```bash
kubectl apply -f acm/service-mesh/namespaces/
```

This creates:

- **Policy** (`istio-namespaces`) — contains a ConfigurationPolicy that enforces the `istio-system` and `istio-cni` namespaces exist.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy -n istio-policies
```

All targeted clusters should show `Compliant` once the namespaces have been created.

## Certificate distribution

### Step 6: Create the CA certificates and distribute to managed clusters

Apply the cert-manager resources and ACM policy that create and distribute the Istio CA certificate:

```bash
kubectl apply -f acm/service-mesh/certificates/
```

This creates the following cert-manager resources on the hub cluster:

- **ClusterIssuer** (`selfsigned`) — a self-signed issuer used to bootstrap the root CA.
- **Certificate** (`root-ca`) — a self-signed root CA certificate used for signing.
- **Issuer** (`root-ca`) — a CA issuer that references the root CA certificate.
- **Certificate** (`istio-ca`) — an intermediate CA certificate for Istio, signed by the root CA.

And the following ACM resources to distribute the certificate:

- **Policy** (`istio-ca-certificate`) — contains a ConfigurationPolicy that creates the `cacerts` Secret in `istio-system` on managed clusters, using hub templates to pull certificate data from the hub.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy.

Verify the certificates are ready:

```bash
kubectl get certificates -n istio-policies
```

Verify policy compliance:

```bash
kubectl get policy istio-ca-certificate -n istio-policies
```

## Istio installation

### Step 7: Install Istio on managed clusters

Apply the Policy and PlacementBinding that create the IstioCNI and Istio resources on all mesh clusters:

```bash
kubectl apply -f acm/service-mesh/istio/
```

This creates:

- **Policy** (`istio`) — contains a ConfigurationPolicy that enforces an `IstioCNI` resource in the `istio-cni` namespace and an `Istio` resource in the `istio-system` namespace.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy istio -n istio-policies
```

Verify Istio is running on managed clusters:

```bash
kubectl get istiocni -A
kubectl get istio -A
```

## East-west gateway

### Step 8: Deploy the east-west gateway on managed clusters

Apply the Policy and PlacementBinding that create a Kubernetes Gateway for cross-network traffic on all mesh clusters:

```bash
kubectl apply -f acm/service-mesh/east-west-gateway/
```

This creates:

- **Policy** (`east-west-gateway`) — contains a ConfigurationPolicy that enforces a Gateway resource in `istio-system` using the `istio` GatewayClass, with a TLS passthrough listener on port 15443 for `*.local` hostnames. The gateway is labeled with `topology.istio.io/network` set to `network-<clusterName>`.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy east-west-gateway -n istio-policies
```

Verify the gateway is running on managed clusters:

```bash
kubectl get gateways -n istio-system
```

## Managed Service Accounts

### Step 9: Create a ManagedServiceAccount per managed cluster

Apply the Policy that creates a ManagedServiceAccount in each managed cluster's namespace on the hub. The policy is bound to `local-cluster` so the ConfigurationPolicy runs on the hub itself. It uses `namespaceSelector` to match namespaces that are both managed cluster namespaces (`cluster.open-cluster-management.io/managedCluster` exists) and labeled `mesh=enabled`:

```bash
kubectl apply -f acm/service-mesh/managed-service-accounts/
```

Verify policy compliance:

```bash
kubectl get policy managed-service-account -n istio-policies
```

## Istio reader ClusterRoleBinding

### Step 10: Bind the ManagedServiceAccount to the istio-reader ClusterRole

Apply the Policy and PlacementBinding that create a ClusterRoleBinding on all mesh clusters, granting the `istio-reader-service-account` ManagedServiceAccount the `istio-reader-clusterrole-istio-system` ClusterRole:

```bash
kubectl apply -f acm/service-mesh/reader-clusterrolebinding/
```

This creates:

- **Policy** (`istio-reader-clusterrolebinding`) — contains a ConfigurationPolicy that enforces a ClusterRoleBinding binding the ManagedServiceAccount to the istio-reader ClusterRole.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy istio-reader-clusterrolebinding -n istio-policies
```

## Remote secret distribution

### Step 11: Distribute Istio remote secrets across clusters

Apply the Policy that distributes Istio remote secrets for multi-cluster communication. The policy is bound to the `local-cluster` Placement so the ConfigurationPolicy runs on the hub. It uses managed cluster templates with `object-templates-raw` to:

1. Range over all ManagedClusters labeled `mesh=enabled`.
2. For each target cluster, create a ManifestWork in its namespace on the hub.
3. For each other mesh cluster (source), look up the MSA token secret and API server URL.
4. Construct a kubeconfig-style Secret and include it in the ManifestWork manifests.

The ManifestWork agent on each managed cluster then creates the remote secrets in `istio-system`. This ensures that cluster A gets remote secrets for cluster B and C (and vice versa), enabling Istio to discover services across clusters.

```bash
kubectl apply -f acm/service-mesh/remote-secrets/
```

Verify policy compliance:

```bash
kubectl get policy istio-remote-secrets -n istio-policies
```

Verify the remote secrets were created on the managed clusters (one secret per remote cluster):

```bash
kubectl get secrets -n istio-system -l istio/multiCluster=true
```

## Verification with sample applications

### Step 12: Label managed clusters with mesh-version

Label one cluster as `v1` and another as `v2`. The `v1` cluster will run helloworld-v1 and curl, while the `v2` cluster will run helloworld-v2:

```bash
kubectl label managedcluster <cluster-1-name> mesh-version=v1
kubectl label managedcluster <cluster-2-name> mesh-version=v2
```

### Step 13: Create the Placements for v1 and v2 clusters

Apply the Placements that select clusters by `mesh-version` label:

```bash
kubectl apply -f acm/applications/placement/
```

### Step 14: Create the sample namespace

Apply the namespace policy to create the `sample` namespace with Istio sidecar injection enabled on all mesh clusters:

```bash
kubectl apply -f acm/applications/namespace/
```

Verify policy compliance:

```bash
kubectl get policy sample-namespace -n istio-policies
```

### Step 15: Deploy the helloworld application

Apply the helloworld policies. This creates the helloworld Service on all mesh clusters, then deploys helloworld-v1 to v1 clusters and helloworld-v2 to v2 clusters:

```bash
kubectl apply -f acm/applications/helloworld/
```

Verify policy compliance:

```bash
kubectl get policy -n istio-policies | grep helloworld
```

### Step 16: Deploy the curl client

Apply the curl policy to deploy the curl client on v1 clusters:

```bash
kubectl apply -f acm/applications/curl/
```

Verify policy compliance:

```bash
kubectl get policy curl -n istio-policies
```

### Step 17: Verify multicluster connectivity

From the curl pod on the v1 cluster, send requests to the helloworld service. You should see responses from both v1 and v2:

```bash
kubectl exec -n sample -c curl "$(kubectl get pod -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello
```
