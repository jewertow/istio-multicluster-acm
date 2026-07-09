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

## Namespace creation

### Step 4: Ensure the istio-system namespace on managed clusters

Apply the Policy and PlacementBinding that enforce the `istio-system` namespace on all mesh clusters:

```bash
kubectl apply -f acm/namespaces/
```

This creates:

- **Policy** (`istio-system-namespace`) — contains a ConfigurationPolicy that enforces the `istio-system` namespace exists.
- **PlacementBinding** — binds the `mesh-clusters` Placement to the Policy so it is applied to the selected clusters.

Verify policy compliance:

```bash
kubectl get policy -n istio-policies
```

All targeted clusters should show `Compliant` once the namespace has been created.

## Certificate distribution

### Step 5: Create the CA certificates and distribute to managed clusters

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

## Managed Service Accounts

### Step 6: Create a ManagedServiceAccount per managed cluster

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
