# TritonBench Build Infra Configuration on Google Cloud Platform

It defines the specification of infrastruture used by TorchBench CI.
The Infra is a Kubernetes cluster built on top of Google Cloud Platform.


## Step 1: Create the cluster and install the ARC Controller

```

# Get credentials for the cluster so that kubectl could use it
gcloud container clusters get-credentials --location us-central1 tritonbench-build-cluster

# Install the ARC controller
NAMESPACE="arc-systems"
helm install arc \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

To upgrade:

```
helm upgrade --install arc -n arc-systems oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

To view the logs:

```
# first, get pod-name
kubectl get pods -n arc-systems
# then, view the pod logs
kubectl logs -n arc-systems <pod-name>
```

## Step 2: Create secrets and assign it to the namespace

The secrets need to be added to both `arc-systems` and `arc-runners` namespaces.

```
# Set GitHub App secret
kubectl create secret generic arc-secret \
   --namespace=arc-runners \
   --from-literal=github_app_id=<GITHUB_APP_ID> \
   --from-literal=github_app_installation_id=<GITHUB_APP_INSTALL_ID> \
   --from-file=github_app_private_key=<GITHUB_APP_PRIVKEY_FILE>

# Alternatively, set classic PAT
kubectl create secret generic arc-secret \
   --namespace=arc-runners \
   --from-literal=github_token="<GITHUB_PAT>" \
```

To get, delete, or update the secrets:

```
# Get
kubectl get -A secrets
# Delete
kubectl delete secrets -n arc-runners arc-secret
# Update
kubectl edit secrets -n arc-runners arc-secret
```

## Step 3: Install runner scale set

```
INSTALLATION_NAME="build-runner" \
NAMESPACE="arc-runners" \
GITHUB_SECRET_NAME="arc-secret" \
helm install "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    -f values.yaml \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

To upgrade or uninstall the runner scale set:

```
# command to upgrade
helm upgrade --install build-runner -n arc-runners -f ./values.yaml oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set

# command to uninstall
helm uninstall -n arc-runners build-runner
```
