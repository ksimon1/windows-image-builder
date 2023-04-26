# THIS PROJECT IS NOT CONNECTED TO ANY CLUSTER AND WILL NOT BUILD ANY WINDOWS DISK!

# windows-image-builder

Build your own Windows image from ISO file for OpenShift Virtualization and Pipelines as Code!

Before you will start to build your own Windows images, you need to connect your repository with your cluster. You can find how to do it in https://pipelinesascode.com/docs/install/. Don’t forget to create repository CR https://pipelinesascode.com/docs/guide/repositorycrd/#repository-cr.

After you linked your cluster and repository, you have to populate your repository with Pipeline and ConfigMaps and PipelineRuns from this repository. Feel free to do any modifications to the Pipeline.

You need only autounattend xml file and registry, where the result image will be pushed.

Examples of PipelineRuns with preset parameters are located under [.tekton](.tekton).

> [!IMPORTANT]
> Example PipelineRuns have special parameter acceptEula. By setting this parameter, you are agreeing to the applicable 
> Microsoft end user license agreement(s) for each deployment or installation for the Microsoft product(s). In case you 
> set it to false, the Pipeline will exit in the first task.

## Warnings
Be careful when when the result disk already exists and is used. The triggered PipelineRun will replace disks already existing in cluster.

Do not set the same `baseDvName` param in multiple PipelineRuns, otherwise all PipelineRuns will recreate the same DV/DS and original DV/DS will be replaced.

## How to create Windows disk one time

Create new merge request with PipelineRun in `.tekton` folder and ConfigMap with autounattend xml in `configmaps/autounattend` folder. You can find examples of autounattend xml in `configmaps/autounattend` folder (e.g. more advanced configuration with multiple answerfiles and reboots is `windows11-qe-virtio.yaml`)

Open merge request against main branch and wait until the CI finishes the image build. The image should be available in the registry you defined. Once the image is pushed, you can close the PR. If the registry is not public, create new secret in the namespace where is the PAC configured to run the automation with registry credentials in format:

```
apiVersion: v1
kind: Secret
metadata:
  name: disk-pusher-quay
  namespace: windows-builder
type: Opaque
data:
  accessKeyId: <ACCESS_KEY_ID>
  secretKey: <SECRET_KEY>
```
Get the ACCESS_KEY_ID or SECRET_KEY by running: `echo -n "<REGISTRY_USERNAME_OR_PASSWORD>" | base64` and set name of the secret in `secret-name` PipelineRun parameter:

 ```
- name: secret-name
  value: disk-pusher-quay
```

## How to add new Windows version to build automation
There are 2 steps, which needs to be fulfilled.
1) Add configmap to [autounattend folder](configmaps/autounattend).

2) Add new PipelineRun to [.tekton](.tekton). Automation will pickup PipelineRuns from .tekton folder.

The PipelineRun must meet the basic requirements:

`annotations`:

```
pipelinesascode.tekton.dev/pipeline: "pipeline/windows-autounattend.yaml" #define which pipeline automation uses.

pipelinesascode.tekton.dev/on-comment: "^/test-11-no-oobe" # when user comment on MR `/test-11-no-oobe` it will trigger the PipelineRun which have this annotation

pipelinesascode.tekton.dev/on-cel-expression: >-
    (target_branch == "main") &&
    (files.all.exists(x, x.matches('.tekton/window11-no-oobe.yaml|configmaps/autounattend/windows11-noobe.yaml'))) #define rule which will trigger rebuild of the disk when merge request is merged and changes some of the required files
```

`spec.params` - define required parameters for the pipeline.

Other params like `spec.timeouts`, `spec.pipelineRef`, `spec.taskRunSpecs`, `spec.workspaces` should be changed only if necessary.

## Trigerring new builds from PR
Each PipelineRun in [.tekton](.tekton) folder can be triggered from any MR via comment.
Available comments:

`/test-all` - triggers every PipelineRun. !Warning! this will consume many resources on cluster. Use very carefully.

`/test-<Windows version>-<variant>` - e.g. /test-11-no-oobe - triggers only PipelineRun with exact annotation `pipelinesascode.tekton.dev/on-comment: "^/test-11-no-oobe"`
