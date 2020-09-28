# Tasks

A __Task__ is a collection of __Steps__ that you define and arrange in a specific order of execution as part of your continuous integration flow. A __Task__ executes as a Pod on your Kubernetes cluster. A __Task__ is available within a specific namespace, while a __ClusterTask__ is available across the entire cluster.

A Task declaration includes the following elements:

- Parameters
- Resources
- Steps
- Workspaces
- Results

## Defining tasks

A task definition should have the following mandatory fields:

- `apiVersion` - Specifies the API version. For example, `tekton.dev/v1beta1`.
- `kind` - Identifies this resource object as a __Task__ object.
- `metadata` - Specifies metadata that uniquely identifies the __Task__ resource object, like a name.
- `spec` - Specifies the configuration information for this __Task__ resource object.
- `steps` - Specifies one or more container images to run in the __Task__.

### Writing your first task

Create a file `mvn-build-task.yml` with following content:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: mvn
spec:
  workspaces:
  - name: maven-repo
  inputs:
    params:
    - name: GOALS
      description: The Maven goals to run
      type: array
      default: ["package"]
    resources:
    - name: source
      type: git
  steps:
    - name: mvn
      image: gcr.io/cloud-builders/mvn
      workingDir: /workspace/source
      command: ["/usr/bin/mvn"]
      args:
        - -Dmaven.repo.local=$(workspaces.maven-repo.path)
        - "$(inputs.params.GOALS)"
```

This task would build a JAR file for the maven project (yes you guessed it right!) Bryan's Per Store! Given the perfectionist Bryan is, he won't rest nor will let you; until you build a docker image from the jar file. So before you think life is hard, create another file named `docker-image-build.yml` and add the follwoing content to it:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kaniko
spec:
  params:
  - name: IMAGE
    description: Name (reference) of the image to build.
  - name: DOCKERFILE
    description: Path to the Dockerfile to build.
    default: ./Dockerfile
  - name: CONTEXT
    description: The build context used by Kaniko.
    default: ./
  - name: EXTRA_ARGS
    default: ""
  - name: BUILDER_IMAGE
    description: The image on which builds will run
    default: gcr.io/kaniko-project/executor:latest
  workspaces:
  - name: source
  results:
  - name: IMAGE-DIGEST
    description: Digest of the image just built.

  steps:
  - name: build-and-push
    workingDir: $(workspaces.source.path)
    image: $(params.BUILDER_IMAGE)
    # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
    # https://github.com/tektoncd/pipeline/pull/706
    env:
    - name: DOCKER_CONFIG
      value: /tekton/home/.docker
    command:
    - /kaniko/executor
    - $(params.EXTRA_ARGS)
    - --dockerfile=$(params.DOCKERFILE)
    - --context=$(workspaces.source.path)/$(params.CONTEXT)  # The user does not need to care the workspace and the source.
    - --destination=$(params.IMAGE)
    - --oci-layout-path=$(workspaces.source.path)/image-digest
    securityContext:
      runAsUser: 0
  - name: write-digest
    workingDir: $(workspaces.source.path)
    image: gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/imagedigestexporter:v0.11.1
    # output of imagedigestexport [{"key":"digest","value":"sha256:eed29..660","resourceRef":{"name":"myrepo/myimage"}}]
    command: ["/ko-app/imagedigestexporter"]
    args:
    - -images=[{"name":"$(params.IMAGE)","type":"image","url":"$(params.IMAGE)","digest":"","OutputImageDir":"$(workspaces.source.path)/image-digest"}]
    - -terminationMessagePath=image-digested
  - name: digest-to-results
    workingDir: $(workspaces.source.path)
    image: stedolan/jq
    script: |
      cat image-digested | jq -j '.[0].value' | tee /tekton/results/IMAGE-DIGEST
```

Don't be scared, we're utilising the `kaniko` task from Tekton's catalogue (a collection of built-in tasks from Tekton to ease up your lives.) For more on why kaniko [refer to this link](https://github.com/tektoncd/catalog/tree/v1beta1/kaniko)

### ClusterTasks

A __ClusterTask__ is a Task scoped to the entire cluster instead of a single namespace. A __ClusterTask__ behaves identically to a __Task__ and therefore everything in this document applies to both.

>Note: When using a __ClusterTask__, you must explicitly set the kind sub-field in the taskRef field to __ClusterTask__. If not specified, the kind sub-field defaults to __Task__.

Below is an example of a Pipeline declaration that uses a __ClusterTask__:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: demo-pipeline
  namespace: default
spec:
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-push
        kind: ClusterTask
      params: ....
```

### Steps

A __Step__ is a reference to a container image that executes a specific tool on a specific input and produces a specific output. To add Steps to a Task you define a steps field (required) containing a list of desired Steps. The order in which the Steps appear in this list is the order in which they will execute.

The following requirements apply to each container image referenced in a steps field:

- Each container image runs to completion or until the first failure occurs.
- The CPU, memory, and ephemeral storage resource requests will be set to zero, or, if specified, the minimums set through LimitRanges in that Namespace, if the container image does not have the largest resource request out of all container images in the __Task__. This ensures that the Pod that executes the __Task__ only requests enough resources to run a single container image in the __Task__ rather than hoard resources for all container images in the Task at once.

### Reserved directories

There are several directories that all Tasks run by Tekton will treat as special

- `/workspace` - This directory is where resources and workspaces are mounted. Paths to these are available to Task authors via variable substitution
- `/tekton` - This directory is used for Tekton specific functionality:
- `/tekton/results` is where results are written to. The path is available to Task authors via `$(results.name.path)`
There are other subfolders which are implementation details of Tekton and users should not rely on their specific behavior as it may change in the future

### Running scripts within Steps

A step can specify a script field, which contains the body of a script. That script is invoked as if it were stored inside the container image, and any args are passed directly to it.

Note: If the script field is present, the step cannot also contain a command field.

Scripts that do not start with a shebang line will have the following default preamble prepended:

```bash
#!/bin/sh
set -xe
```

You can override this default preamble by prepending a shebang that specifies the desired parser. This parser must be present within that Step's container image.

The example below executes a Bash script:

```yaml
steps:
- image: ubuntu  # contains bash
  script: |
    #!/usr/bin/env bash
    echo "Hello from Bash!"
```

> __Exercise__ Create a task and task that executes a simple bash-script, print the output status of the last executed __TaskRun__ (hint: it's sneaking somewhere in install.md)

### Specifying Params in Steps

You can specify parameters, such as compilation flags or artifact names, that you want to supply to the Task at execution time. Parameters are passed to the Task from its corresponding TaskRun.

Parameter names:

- Must only contain alphanumeric characters, hyphens (-), and underscores (_).
- Must begin with a letter or an underscore (_).
- For example, `fooIs-Bar_` is a valid parameter name, but `barIsBa$` or `0banana` are not.

Each declared parameter has a type field, which can be set to either array or string. array is useful in cases where the number of compilation flags being supplied to a task varies throughout the Task's execution. If not specified, the type field defaults to string. When the actual parameter value is supplied, its parsed type is validated against the type field.

The following example illustrates the use of __Parameters__ in a Task. The Task declares two input parameters named `flags` (of type array) and `someURL` (of type string), and uses them in the `steps.args` list. You can expand parameters of type array inside an existing array using the `star` operator. In this example, flags contains the `star` operator: `$(params.flags[*])`.

> __Note__: Input parameter values can be used as variables throughout the Task by using variable substitution.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: task-with-parameters
spec:
  params:
    - name: flags
      type: array
    - name: someURL
      type: string
  steps:
    - name: build
      image: my-builder
      args: ["build", "$(params.flags[*])", "url=$(params.someURL)"]
```

__Exercise__ Create a task and task that executes a simple bash-script which takes two paramters as input and prints their value, also print the output status of the last executed __TaskRun_
