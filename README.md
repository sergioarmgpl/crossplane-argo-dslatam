# crossplane-argo-dslatam
# Create GKE Cluster
To create a GKE cluster follow the next steps:
1. Create a firewall rule to accept all incoming traffic and call it as allin
2. Create a firewall rule to accept all outgoing traffic and call it allout
3. Create the cluster in GCP
4. Access the cluster
5. Create a service account with the following commands:
```
gcloud iam service-accounts create dslatam --display-name="SA Demo Crossplane"
gcloud iam service-accounts add-iam-policy-binding dslatam@cs-nonprod.iam.gserviceaccount.com --member='serviceAccount:dslatam@cs-nonprod.iam.gserviceaccount.com' --role='roles/owner'


gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} \
    --member=serviceAccount:${GCP_SVC_ACC} \
    --role=roles/container.admin

```
6. Create a key to be used in Crossplane:
```
gcloud iam service-accounts keys create key.json --iam-account=dslatam@cs-nonprod.iam.gserviceaccount.com
```

# Install ArgoCD
1. To install ArgoCD run the fullowing steps:
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
2. Access ArgoCD by running the following command:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
3. Install the CLI local client:
```
brew install argocd
```
4. Get the current admin ArgoCD password:
```
argocd admin initial-password -n argocd
```
5. Login with the admin user and previous password and update it:
```
argocd login --insecure localhost:8080
argocd account update-password --insecure --server localhost:8080
```

# Install Crossplane
To install crossplane follow the next steps:
1. Add the helm repository:
```
helm repo add \
crossplane-stable https://charts.crossplane.io/stable
helm repo update
```
2. Install Crossplane using helm:
```
helm install crossplane \
crossplane-stable/crossplane \
--namespace crossplane-system \
--create-namespace
```
3. Install the GCP Provide:
```
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-gcp-storage
spec:
  package: xpkg.upbound.io/upbound/provider-gcp-storage:v0.35.0
EOF
```
4. Create a secret that contains the credentials
```
kubectl create secret \
generic gcp-secret \
-n crossplane-system \
--from-file=creds=./key.json
```

5. Instance a new GCP provider using the secret:
```
kubectl create ns development
cat <<EOF | kubectl apply -f -
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  projectID: cs-nonprod
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: gcp-secret
      key: creds
EOF
```

# Create the initial Projects and Apps
Create the projects for this demo by running the following command:
```
kubectl apply -f projects/
```


# Use the workflows
1. Create a new application
2. Delete an application