# Kubeflow demo - Yelp restaurant reviews

This repository contains a demonstration of Kubeflow capabilities, suitable for
presentation to public audiences.

The base demo includes the following steps:

1. [Install kubeflow locally](#1-install-kubeflow-locally)
1. [Run training locally](#2-run-training-locally)
1. [Install kubeflow on GKE](#3-install-kubeflow-on-gke)
1. [Run training on GKE](#4-run-training-on-gke)
1. [Create the serving and UI components](#5-create-the-serving-and-ui-components)
1. [Bring up a notebook](#6-bring-up-a-notebook)

## Important! Pre-work

Before completing any of the below steps, follow the instructions in
[./demo_setup](https://github.com/kubeflow/examples/blob/master/demos/yelp_demo/demo_setup/README.md)
to prepare for demonstrating Kubeflow:

* [Create a minikube cluster](https://github.com/kubeflow/examples/blob/master/demos/yelp_demo/demo_setup/README.md#4-create-a-minikube-cluster)
* [Create a GKE cluster](https://github.com/kubeflow/examples/blob/master/demos/yelp_demo/demo_setup/README.md#5-create-a-gke-cluster)

## 1. Create clusters

[Setup your environment](https://github.com/kubeflow/examples/tree/master/demos/yelp_demo/demo_setup#2-set-environment-variables)
or source the base file:

```
cd demo_setup
source kubeflow-demo-base.env
```

Create a minikube cluster:

```
minikube start \
  --cpus 4 \
  --memory 8096 \
  --disk-size=50g \
  --kubernetes-version v1.10.6
```

Create a GKE cluster with access to GPUs and TPUs:

```
gcloud beta container clusters create ${CLUSTER} \
  --project ${DEMO_PROJECT} \
  --zone ${ZONE} \
  --accelerator type=nvidia-tesla-k80,count=2 \
  --cluster-version 1.10.6-gke.2 \
  --enable-ip-alias \
  --enable-tpu \
  --machine-type n1-highmem-8 \
  --scopes cloud-platform,compute-rw,storage-rw \
  --verbosity error
```

## 1. Install kubeflow locally

Run the following script to create a ksonnet app for Kubeflow and deploy it:

```
export KUBEFLOW_VERSION=0.2.5
curl https://raw.githubusercontent.com/kubeflow/kubeflow/v${KUBEFLOW_VERSION}/scripts/deploy.sh | bash
```

View the installed components:

```
kubectl get pod
```

## 2. Run training locally

Retrieve the following files for the t2tcpu & t2ttpu jobs:

```
cd kubeflow_ks_app
cp ${REPO_PATH}/demo/components/t2t[ct]pu.* components
cp ${REPO_PATH}/demo/components/params.* components
```

Set parameter values for training:

```
ks param set --env default t2tcpu \
  cloud "minikube"
ks param set --env default t2tcpu \
  workers 2
```

Generate manifests and apply to cluster:

```
ks apply default -c t2tcpu
```

View the new training pod and wait until it has a `Running` status:

```
kubectl get pod
```

View the logs to watch training commence:

```
kubectl logs -f t2tcpu-master-0 | grep INFO:tensorflow
```

## 3. Install kubeflow on GKE

Switch to a GKE cluster:

```
kubectl config use gke
```

Create an environment:

```
ks env add gke
```

Install kubeflow on the cluster:

```
ks apply gke -c kubeflow-core
```

View the installed components:

```
kubectl get pod
```

## 4. Run training on GKE

### Distributed CPU training

Set parameter values for training:

```
ks param set --env gke t2tcpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_CPU}
```

Warning: this command can take hours(!) to complete if the number of steps is
above 1000.

```
ks apply gke -c t2tcpu
```

View the new training pod and wait until it has a `Running` status:

```
kubectl get pod
```

View the logs to watch training commence:

```
kubectl logs -f t2tcpu-master-0 | grep INFO:tensorflow
```

### Distributed TPU training

Set parameter values for training:

```
ks param set --env gke t2ttpu \
  outputGCSPath ${GCS_TRAINING_OUTPUT_DIR_TPU}
```

Warning: this command can take hours(!) to complete if the number of steps is
above 1000.

```
ks apply gke -c t2ttpu
```

Verify that a TPU is being provisioned by viewing pod status. It should remain
in Pending state for 3-4 minutes with the message
`Creating Cloud TPUs for pod default/t2ttpu-master-0`.

```
kubectl describe pod t2ttpu-master-0
```

Once it has `Running` status, view the logs to watch training commence:

```
kubectl logs -f t2ttpu-master-0 | grep INFO:tensorflow
```

## 5. Create the serving and UI components

Retrieve the following files for the serving & UI components:

```
cp ${REPO_PATH}/demo/components/serving.* components
cp ${REPO_PATH}/demo/components/ui.* components
```

Create the serving and UI components:

```
ks apply gke -c serving -c ui
```

Forward the port for viewing from a local browser:
```
UI_POD=$(kubectl get po -l app=kubeflow-demo-ui | \
  grep kubeflow-demo-ui | \
  head -n 1 | \
  cut -d " " -f 1 \
)
kubectl port-forward ${UI_POD} 8080:80
```

Optional: If necessary, setup an SSH tunnel from your local laptop into the
compute instance connecting to GKE:

```
ssh $HOST -L 8080:localhost:8080
```

To show the naive version, navigate to [localhost:8080](localhost:8080) from a browser.

To show the ML version, navigate to
[localhost:8080/kubeflow](localhost:8080/kubeflow) from a browser.

## 6. Bring up a notebook

Connect to the Central Dashboard by forwarding a port to one of the ambassador
pods:

```
AMBASSADOR_POD=$(kubectl get po -l service=ambassador | \
  grep ambassador | \
  head -n 1 | \
  cut -d " " -f 1 \
)
kubectl port-forward ${AMBASSADOR_POD} 8081:80
```

Open a browser and connect to [localhost:8081](localhost:8081).
Show the TF-job dashboard, then click on Jupyterhub.
Log in with any username and password combination and wait until the page
refreshes. Spawn a new pod with these resource requirements:

| Resource              | Value                                                                |
| --------------------- | -------------------------------------------------------------------- |
| Image                 | `gcr.io/kubeflow-images-public/tensorflow-1.7.0-notebook-gpu:v0.2.1` |
| CPU                   | 2                                                                    |
| Memory                | 48G                                                                  |
| Extra Resource Limits | `{"nvidia.com/gpu":2}`                                               |

Once the notebook environment is
available, open a new terminal and upload this
[Yelp notebook](notebooks/yelp.ipynb).

Execute the notebook to show that it works.

