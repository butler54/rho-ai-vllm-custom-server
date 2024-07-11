# rhoai-kserve-from-pvc

[Open Data Hub](https://opendatahub.io/) and [Red Hat OpenShift AI](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-ai) both use Kserve as the infrastructure for serving models.

The default setup is that Kserve is configured to pull the models from an s3 compatible store (e.g AWS S3, IBM COS, Minio, Red Hat MultiCloudGateway etc.).

The process that occurs is that when Kserve initializes `InferenceService` runtime's pod.
An init container downloads the model into an [`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) volume mount.

This enables the inference server to exploit the peformance of locally attached storage if the server crashes as needs to restart.

However, some environments are constrained:

- There is insufficient local storage to cache the model on local disk.
  - Filling local storage on nodes can have adverse side effects including thrashing as the node tries to eject enough workload to obtain the minimum required free space

- S3 is not universal on-premise.
   - While services such as Minio or NooBaa Multi Cloud Gateway can be deployed on the cluster, in the end they will then be consuming from PVCs.

As an alternative this repository documents the process for using a PVC for storage of a model and therefore avoiding requiring s3 and local `emptyDir` volume mounts


## Caveats

- At this stage the repository provides an *example* of doing so not a customized pipeline.
  - Minimal templating has been used.

- The example was built on a Red Hat OpenShift on AWS cluster. There are references to AWS which may not apply (such as gp3 st)

- This has only being tested with a "Single Model Server" such as vLLM


## Ambitions
- Convert the process into a templated workflow.

# Process

```mermaid
flowchart TD

startNode((Start))


endNode(('Done'))

startNode --> endNode

```


## Once per cluster
In order to serve a model from a PVC the storage class must have a `RECLAIMPOLICY` of `Retain`.

In the case of ths cluster none of the storage classes supported the appropriate reclaim policy (see below):

```shell
oc get sc

NAME             PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2              kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   43h
gp2-csi          ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   42h
gp3 (default)    ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   43h
gp3-csi          ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   42h
```

However, reclaim is a cluster side controlled policy (and not a feature controlled by providers):

- Pull down the closest storage class you want to use: `oc get sc gp3-csi -o yaml > .sampleManifests/original-gp3-storage-class.yaml`
- Copy to a clean file. Remove `uuid` fields and change `reclaimPolicy: Delete` to `reclaimPolicy: Retain`
- Apply the new manifest `oc apply -f ./sampleMainfest/gp3-sc-with-retain.yaml`

The result should be an additional storage class:

```shell
oc get sc
NAME             PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2              kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   true                   43h
gp2-csi          ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   43h
gp3 (default)    ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   43h
gp3-csi          ebs.csi.aws.com         Delete          WaitForFirstConsumer   true                   43h
gp3-csi-retain   ebs.csi.aws.com         Retain          WaitForFirstConsumer   true                   159m
```

## Approximately Once per model

**This section is assumed to be run from your notebook server on the cluster with**

### Preload a pvc with your model

- `oc login` to the cluster in your notebook e.g. `!oc login --token .. `

- Create a [pvc](./sampleManifests/model-pvc.yaml) and apply the pvc (named: `vllm-model-cache`)
   -  `oc apply -f ./sampleManifest/model-pvc.yaml`

- Create a pod which does nothing, and contains `tar` to enable the copy 
  - `oc apply -f ./sampleManifest/model-storage-pod.yaml`

- Use `oc cp` to copy data up to the pvc
  - `oc cp granite-7b-lab model-store-pod-ubi:/pv/granite-7b-lab -c model-store`

- In the case above, `granite-7b-lab` is the relative path to the model in the notebook

- Delete the pod but not the pvc
  - `oc delete -f ./sampleManifest/model-storage-pod.yaml`

The model is now in a pvc ready to serve!

### Deploying the customized model server

When deploying a single model server in ODH / Red Hat OpenShift AI two objects are created:

1. A `servingRuntime`
2. A `servingInstance`

The easiest way to get valid objects is to attempt to deploy a model using the UI first.
This example deployed a vLLM runtime called `granite` to the `llm` namespace.

With this the objects are retrievalble 

```shell
oc get servingruntimes.serving.kserve.io -n llm granite -o yaml > ./sampleManifests/granite-servingRuntime.yaml

oc get inferenceservices.serving.kserve.io -n llm granite -o yaml > ./sampleManifests/granite-instanceServices.yaml
```

#### Changes required

- The `servingRuntime` purely needs name references changed. E.g. in this case find `granite` and replace with `granite-pvc`
  - new file is [here](./sampleManifests/new-granite-pvc-servingRuntime.yaml)

- The `instanceServices` need:
  - A name change from `granite` to `granite-pvc`. The `runtime:` element *MUST* match the name of the `servingRuntime`
- The storage needs to be changed

Original storage
```yaml
  ...
      runtime: granite
      storage:
        key: aws-connection-minio
        path: granite-7b-lab
  ...
```

New storage

```yaml
  ...
      runtime: granite-pvc
      storageUri: pvc://vllm-model-cache/granite-7b-lab
  ...
```

full file is [here](sampleManifests/new-granite-pvc-instanceServices.yaml)

The PVC must be in the same namespace.

This new model can be applied by:

```shell
oc apply -f ./sampleManifests/new-granite-pvc-servingRuntime.yaml
oc apply -f ./sampleManifests/new-granite-pvc-instanceServices.yaml
```

Inspecting `/var/lib/kubelet/pods` should show that pods do not use local `emptyDir` storage.
