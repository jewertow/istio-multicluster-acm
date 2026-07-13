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
kubectl apply -f acm/placement/
```

## Service Mesh operator

### Step 4: Install the OpenShift Service Mesh 3 operator on managed clusters

Apply the Policy and PlacementBinding that install the Service Mesh operator via OLM on all mesh clusters:

```bash
kubectl apply -f acm/servicemesh-operator/
```

This creates:

- **Policy** (`servicemeshoperator`) — contains a ConfigurationPolicy that enforces a Subscription for the `servicemeshoperator3` package from the `redhat-operators` catalog.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy servicemeshoperator -n istio-policies
```

Verify the operator is installed on managed clusters:

```bash
kubectl get csv -n openshift-operators | grep servicemeshoperator
```

## Namespace creation

### Step 5: Ensure the istio-system and istio-cni namespaces on managed clusters

Apply the Policy and PlacementBinding that enforce the `istio-system` and `istio-cni` namespaces on all mesh clusters:

```bash
kubectl apply -f acm/namespaces/
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
kubectl apply -f acm/certificates/
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
kubectl apply -f acm/istio/
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

## Managed Service Accounts

### Step 8: Create a ManagedServiceAccount per managed cluster

Apply the Policy that creates a ManagedServiceAccount in each managed cluster's namespace on the hub. The policy is bound to `local-cluster` so the ConfigurationPolicy runs on the hub itself. It uses `namespaceSelector` to match namespaces that are both managed cluster namespaces (`cluster.open-cluster-management.io/managedCluster` exists) and labeled `mesh=enabled`:

```bash
kubectl apply -f acm/managed-service-accounts/
```

Verify policy compliance:

```bash
kubectl get policy managed-service-account -n istio-policies
```

Verify the ManagedServiceAccounts were created (one per managed cluster):

```bash
kubectl get managedserviceaccounts -A
```

## Remote secret distribution

### Step 9: Distribute Istio remote secrets across clusters

Apply the Policy that distributes Istio remote secrets for multi-cluster communication. The policy is bound to the `local-cluster` Placement so the ConfigurationPolicy runs on the hub. It uses managed cluster templates with `object-templates-raw` to:

1. Range over all ManagedClusters labeled `mesh=enabled`.
2. For each target cluster, create a ManifestWork in its namespace on the hub.
3. For each other mesh cluster (source), look up the MSA token secret and API server URL.
4. Construct a kubeconfig-style Secret and include it in the ManifestWork manifests.

The ManifestWork agent on each managed cluster then creates the remote secrets in `istio-system`. This ensures that cluster A gets remote secrets for cluster B and C (and vice versa), enabling Istio to discover services across clusters.

```bash
kubectl apply -f acm/remote-secrets/
```

Verify policy compliance:

```bash
kubectl get policy istio-remote-secrets -n istio-policies
```

Verify the remote secrets were created on the managed clusters (one secret per remote cluster):

```bash
kubectl get secrets -n istio-system -l istio/multiCluster=true
```
