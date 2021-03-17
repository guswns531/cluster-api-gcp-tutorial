# cluster-api-gcp-tutorial
cluster-api-gcp-tutorial

```bash
# Login glcoud & Create New Project
gcloud init

# Check Project ID
gcloud config get-value project

# Enable and Set up Compute API 
gcloud services enable compute.googleapis.com
# gcloud services list --available | grep compute

# Setup default region & zone
gcloud compute project-info add-metadata \
    --metadata google-compute-default-region=asia-northeast3,google-compute-default-zone=asia-northeast3-a

# Check Compute API
gcloud compute project-info describe --project jang-project-1

# Create Service Account 
gcloud iam service-accounts create jang-service-account-1

# Role Binding as project owner
gcloud projects add-iam-policy-binding jang-project-1 \
    --member="serviceAccount:jang-service-account-1@jang-project-1.iam.gserviceaccount.com" \
    --role="roles/owner"

# Get Service acoount Json Key
gcloud iam service-accounts keys create ./service-account-key.json \
  --iam-account jang-service-account-1@jang-project-1.iam.gserviceaccount.com

export GCP_PROJECT_ID=jang-project-1 
export GOOGLE_APPLICATION_CREDENTIALS=~/cluster-api/tutorial/service-account-key.json

# It's for make build-gce-ubuntu-1804 

export  PATH="/Users/hyeonjunjang/cluster-api/tutorial/image-builder/images/capi/.local/bin:${PATH}"
export PATH="/Users/hyeonjunjang/Library/Python/3.8/bin:${PATH}"

# Clone the image builder repository if you haven't already.
git clone https://github.com/kubernetes-sigs/image-builder.git image-builder

# Modify Kuberenetes Version : Defualt 1.18.15
vi /image-builder/images/capi/packer/config/kubernetes.json

# Change directory to images/capi within the image builder repository
cd image-builder/images/capi

# Run the Make target to generate GCE images.
make build-gce-ubuntu-1804

# Check that you can the published images.
gcloud compute images list --project ${GCP_PROJECT_ID} --no-standard-images --filter="family:capi-ubuntu-1804-k8s"

export GCP_PROJECT=jang-project-1 
export GOOGLE_APPLICATION_CREDENTIALS=~/cluster-api/tutorial/service-account-key.json

git clone https://github.com/kubernetes-sigs/cluster-api-provider-gcp.git

cd cluster-api-provider-gcp

# Build controller image
make docker-build

# Enable GCP Container Registry
gcloud services enable containerregistry.googleapis.com

# Login Registry
docker login -u _json_key -p "$(cat ~/cluster-api/tutorial/service-account-key.json)" https://gcr.io/jang-project-1/

# Push image to GCP Container Registry
make docker-push
# docker push gcr.io/jang-project-1/cluster-api-gcp-controller-amd64:dev

# Configuring public access to images
gsutil iam ch allUsers:objectViewer $(gsutil ls)
#gsutil iam ch allUsers:objectViewer gs://artifacts.jang-project-1.appspot.com/

export GCP_B64ENCODED_CREDENTIALS=$( cat ${GOOGLE_APPLICATION_CREDENTIALS} | base64 | tr -d '\n' )

# Create Management Cluster
make create-management-cluster

export GCP_PROJECT=jang-project-1
export GCP_NETWORK_NAME=jang-network-1
export GCP_REGION=asia-northeast3
export CLUSTER_NAME=jang-cluster-1
export KUBERNETES_VERSION=v1.18.1
export CONTROL_PLANE_MACHINE_COUNT=1
export WORKER_MACHINE_COUNT=1
export GCP_CONTROL_PLANE_MACHINE_TYPE=e2-standard-2
export GCP_NODE_MACHINE_TYPE=e2-standard-2

# Create Workload Cluster
make create-workload-cluster

# Delete Workload Cluster
kubectl delete cluster $(CLUSTER_NAME)

kubectl delete cluste jang-cluster-1

# Delete Management Cluster
kind delete cluster --name=clusterapi

# Revoking permissions to access images
gsutil iam ch -d allUsers $(gsutil ls)
#gsutil iam ch -d allUsers gs://artifacts.jang-project-1.appspot.com/
```

### Using Clusterctl

```bash
# Create kind management cluster.
kind create cluster --name=<cluster name> 
# > kind create cluster --name=cluster

# CAPI Controller 설치
clusterctl init --infrastructure gcp

# Get Cluster Env value 
clusterctl config cluster --infrastructure gcp --list-variables <cluster name> 
# > clusterctl config cluster --infrastructure gcp --list-variables cluster

# Create Worload Cluster environment variables
clusterctl config cluster --config cluster-config.yml <cluster name> > cluster.yaml
# >  clusterctl config cluster --config cluster-config.yml cluster > cluster.yaml

kubectl apply -f cluster.yaml

# Get kubeconfig
clusterctl get kubeconfig cluster > cluster-kubeconfig

# Deploy calico
kubectl --kubeconfig cluster-kubeconfig apply -f https://docs.projectcalico.org/manifests/calico.yaml
```





## **Inspect Makefile for Management-Cluster**

```bash
## Create kind management cluster.
kind create cluster --name=clusterapi

# Install cert manager and wait for availability
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v0.11.1/cert-manager.yaml
kubectl wait --for=condition=Available --timeout=5m apiservice v1beta1.webhook.cert-manager.io

# Deploy CAPI
wget -O- https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.3.10/cluster-api-components.yaml | $(ENVSUBST) | kubectl apply -f -
# wget -O- https://github.com/kubernetes-sigs/cluster-api/releases/download/v0.3.12/cluster-api-components.yaml | /Users/hyeonjunjang/cluster-api/tutorial/cluster-api-provider-gcp/hack/tools/bin/envsubst | kubectl apply -f -

# Deploy CAPG
kind load docker-image $(CONTROLLER_IMG)-$(ARCH):$(TAG) --name=clusterapi
# kind load docker-image gcr.io/jang-project-1/cluster-api-gcp-controller-amd64:dev --name=clusterapi
$(KUSTOMIZE) build config | $(ENVSUBST) | kubectl apply -f -
# /Users/hyeonjunjang/cluster-api/tutorial/cluster-api-provider-gcp/hack/tools/bin/kustomize-v3.5.4 build config | /Users/hyeonjunjang/cluster-api/tutorial/cluster-api-provider-gcp/hack/tools/bin/envsubst | kubectl apply -f -


# Wait for CAPI pods
kubectl wait --for=condition=Ready --timeout=5m -n capi-system pod -l cluster.x-k8s.io/provider=cluster-api
kubectl wait --for=condition=Ready --timeout=5m -n capi-kubeadm-bootstrap-system pod -l cluster.x-k8s.io/provider=bootstrap-kubeadm
kubectl wait --for=condition=Ready --timeout=5m -n capi-kubeadm-control-plane-system pod -l cluster.x-k8s.io/provider=control-plane-kubeadm

# Wait for CAPG pods
kubectl wait --for=condition=Ready --timeout=5m -n capg-system pod -l cluster.x-k8s.io/provider=infrastructure-gcp

# required sleep for when creating management and workload cluster simultaneously
sleep 10
@echo 'Set kubectl context to the kind management cluster by running "kubectl config set-context kind-clusterapi"'


```





## **Inspect Makefile for Workload-Cluster**

```bash

# Create workload Cluster.
$(KUSTOMIZE) build templates | $(ENVSUBST) | kubectl apply -f -
# /Users/hyeonjunjang/cluster-api/tutorial/cluster-api-provider-gcp/hack/tools/bin/kustomize-v3.5.4 build templates | /Users/hyeonjunjang/cluster-api/tutorial/cluster-api-provider-gcp/hack/tools/bin/envsubst | kubectl apply -f -


# Wait for the kubeconfig to become available.
${TIMEOUT} 5m bash -c "while ! kubectl get secrets | grep $(CLUSTER_NAME)-kubeconfig; do sleep 1; done"
# /usr/local/bin/timeout 5m bash -c "while ! kubectl get secrets | grep jang-cluster-1-kubeconfig; do sleep 1; done"

# Get kubeconfig and store it locally.
kubectl get secrets $(CLUSTER_NAME)-kubeconfig -o json | jq -r .data.value | base64 --decode > $(CAPG_WORKER_CLUSTER_KUBECONFIG)
#kubectl get secrets jang-cluster-1-kubeconfig -o json | jq -r .data.value | base64 --decode > "/tmp/kubeconfig"

${TIMEOUT} 15m bash -c "while ! kubectl --kubeconfig=$(CAPG_WORKER_CLUSTER_KUBECONFIG) get nodes | grep master; do sleep 1; done"
# /usr/local/bin/timeout 15m bash -c "while ! kubectl --kubeconfig="/tmp/kubeconfig" get nodes | grep master; do sleep 1; done"

# Deploy calico
kubectl --kubeconfig=$(CAPG_WORKER_CLUSTER_KUBECONFIG) apply -f https://docs.projectcalico.org/manifests/calico.yaml
# kubectl --kubeconfig=/tmp/kubeconfig apply -f https://docs.projectcalico.org/manifests/calico.yaml

@echo 'run "kubectl --kubeconfig=$(CAPG_WORKER_CLUSTER_KUBECONFIG) ..." to work with the new target cluster'
```

**Build GCP VM Image**

## Summary environment variables

```bash
export GCP_PROJECT_ID=jang-project-1 
export GOOGLE_APPLICATION_CREDENTIALS=~/cluster-api/tutorial/service-account-key.json
export GCP_B64ENCODED_CREDENTIALS=$( cat ${GOOGLE_APPLICATION_CREDENTIALS} | base64 | tr -d '\n' )
export GCP_PROJECT=jang-project-1
export GCP_NETWORK_NAME=jang-network-1
export GCP_REGION=asia-northeast3
export CLUSTER_NAME=jang-cluster-1
export KUBERNETES_VERSION=v1.18.1
export CONTROL_PLANE_MACHINE_COUNT=1
export WORKER_MACHINE_COUNT=1
export GCP_CONTROL_PLANE_MACHINE_TYPE=e2-standard-2
export GCP_NODE_MACHINE_TYPE=e2-standard-2
```

Cluster-config.yml

```yaml
GCP_PROJECT_ID: jang-project-1 
GOOGLE_APPLICATION_CREDENTIALS: ~/cluster-api/tutorial/service-account-key.json
GCP_B64ENCODED_CREDENTIALS: <GCP_B64ENCODED_CREDENTIALS>
GCP_PROJECT: jang-project-1
GCP_NETWORK_NAME: jang-network-1
GCP_REGION: asia-northeast3
CLUSTER_NAME: jang-cluster-1
KUBERNETES_VERSION: v1.17.1
CONTROL_PLANE_MACHINE_COUNT: 1
WORKER_MACHINE_COUNT: 1
GCP_CONTROL_PLANE_MACHINE_TYPE: e2-standard-2
GCP_NODE_MACHINE_TYPE: e2-standard-2
```

