# ml-on-gke-demo
Demo for runing ML jobs on GKE using dynamic resources

Here goes....

### Step 0: High level configuration
```
PROJECT=ashires-joonix-test-1
CLUSTER_NAME=ray-test-cluster-3
ZONE=us-central1-b
NETWORK=data-vpc
SUBNETWORK=us-dev
GPU_POOL="gpu-pool-2"
DWS_POOL="dws-pool-3"
WORKLOAD_IDENTITY_STR="ashires-joonix-test-1.svc.id.goog"
```
TODO: setup workload identity and create workload identity pool for GCSFUSE

### Step 1: Create a GKE cluster
```
gcloud beta container --project $PROJECT \
 clusters create $CLUSTER_NAME --zone $ZONE \
 --network $NETWORK --subnetwork $SUBNETWORK \
 --enable-private-nodes --enable-master-global-access --enable-ip-alias \
 --machine-type "e2-standard-4" --enable-autoscaling --min-nodes "0" --max-nodes "100" \
 --enable-dataplane-v2 --no-enable-master-authorized-networks \
 --addons HorizontalPodAutoscaling,HttpLoadBalancing,NodeLocalDNS,GcePersistentDiskCsiDriver,GcpFilestoreCsiDriver,GcsFuseCsiDriver \
 --node-locations "us-central1-a","us-central1-b","us-central1-c","us-central1-f" \
  --workload-pool $WORKLOAD_IDENTITY_STR
```
### Step 2: Create a GPU-specific nodepool
```
gcloud beta container --project $PROJECT node-pools create $GPU_POOL --cluster $CLUSTER_NAME \
  --zone $ZONE  --machine-type "g2-standard-4" --accelerator "type=nvidia-l4,count=1" \
  --enable-autoscaling --min-nodes "0" --max-nodes "100"  --location-policy "BALANCED" \
  --node-locations "us-central1-a","us-central1-b","us-central1-c"
```
### Step 3: get credentials
```
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE --project $PROJECT
```
### Step 4:install Ray
```
helm repo add kuberay https://ray-project.github.io/kuberay-helm/
helm install kuberay-operator kuberay/kuberay-operator --version 1.1.1
helm install raycluster kuberay/ray-cluster --version 1.1.1
```
### Step 5: install Kubeflow
install Kubeflow Training Operator
```
kubectl apply -k "github.com/kubeflow/training-operator/manifests/overlays/standalone?ref=v1.7.0"
```

### Step 6: install Kueue
```
kubectl apply --server-side -f https://github.com/kubernetes-sigs/kueue/releases/download/v0.8.0/manifests.yaml

```
### Step 7: Setup local environment
install ray client on local environment
```
pip install ray[default]
```
now we can try to submit a Ray job
```
export HEAD_POD=$(kubectl get pods --selector=ray.io/node-type=head -o custom-columns=POD:metadata.name --no-headers)
kubectl exec -it $HEAD_POD -- python -c "import ray; ray.init(); print(ray.cluster_resources())"
```
### Step 8: forward the Ray UI and do the same
port forward the interface
```
kubectl port-forward service/raycluster-kuberay-head-svc 8265:8265
```
open http://localhost:8265 in your web browser 

submit job via interface from the command line
```
ray job submit --address http://localhost:8265 -- python -c "import ray; ray.init(); print(ray.cluster_resources())"
```

### Step 10: Set up Kueue

Keueue basics: need some namespaces for a team
```
kubectl create namespace team-1-namespace
kubectl apply -f kueue-setup.yaml
```
Save the configuration to a YAML file 

Submit an example job to the queue for CPU and GPU resources 
```
kubectl create -f 1-cpu-kueue-job.yaml
kubectl create -f 2-gpu-kueue-job.yaml
kubectl create -f 3-ray-kueue-job.yaml
```

### Step 11: Set up DWS resources

```
gcloud beta container node-pools create $DWS_POOL \
--cluster=$CLUSTER_NAME \
--zone=$ZONE \
--enable-queued-provisioning \
--accelerator type=nvidia-l4,count=1,gpu-driver-version=default \
--machine-type=g2-standard-4 \
--enable-autoscaling  \
--num-nodes=0   \
--total-max-nodes 100  \
--location-policy=ANY \
    --reservation-affinity=none  \
--node-locations "us-central1-a","us-central1-b","us-central1-c" 
```
and the associated kueue resources
```
kubectl apply -f dws-setup.yaml
```

### Step 12: Submit a Ray Job to Kueue to use DWS on-demand GPUs

Create a normal job and a ray example training job
```
kubectl create -f 4-dws-kueue-job.yaml
kubectl create -f 5-ray-dws-kueue-job.yaml
```

### Step 13: Submit a Kubeflow job

Create a pytorch and tensorflow training job
```
kubectl create -f 6-kubeflow-pyt-job.yaml
kubectl create -f 7-kubeflow-tf-job.yaml
```

# Summary

At end of this exercise you have
* created a GKE cluster
* created a node pool for GPUs
* created a node pool for dynamic provisioning of GPUs
* installed Kubeflow, Ray, Kueue on the GKE cluster
* configured queues to manage job submission to node pools
* submitted a variety of jobs to explore the different configurations possible
