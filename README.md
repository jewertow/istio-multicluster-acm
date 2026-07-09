# Istio Multi-Cluster GitOps

Deploy Istio across a fleet of OpenShift/Kubernetes clusters using ACM (Advanced Cluster Management) and ArgoCD.

## Prerequisites

- OpenShift or Kubernetes cluster with ACM hub installed
- `kubectl` CLI configured to access the ACM hub cluster
- Managed clusters imported into ACM

## Step 1: Label managed clusters

Label each cluster that should join the Istio mesh:

```bash
kubectl label managedcluster <cluster-name> mesh=enabled
```

Verify the label:

```bash
kubectl get managedclusters -l mesh=enabled
```

## Step 2: Create the policy namespace

Create a namespace on the hub cluster to hold ACM policies:

```bash
kubectl create namespace istio-policies
```

## Step 3: Create the Placement for mesh clusters

Apply the Placement and ManagedClusterSetBinding. The ManagedClusterSetBinding grants the `istio-policies` namespace access to the `default` ManagedClusterSet. The Placement selects managed clusters with the label `mesh=enabled` and is shared by all ACM policies targeting mesh clusters (namespaces, certificates, etc.):

```bash
kubectl apply -f acm/placement/
```

## Step 4: Ensure the istio-system namespace on managed clusters

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
