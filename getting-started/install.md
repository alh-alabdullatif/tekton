# Installing Tekton

To install the core component of Tekton, Tekton Pipelines, run the command below:

`kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml`

It may take a few moments before the installation completes. You can check the progress with the following command:

`kubectl get pods --namespace tekton-pipelines`

Confirm that every component listed has the status Running.

## Persistent volumes

To run a CI/CD workflow, you need to provide Tekton a Persistent Volume for storage purposes. Tekton requests a volume of 5Gi with the default storage class by default. Your Kubernetes cluster, such as one from Google Kubernetes Engine, may have persistent volumes set up at the time of creation, thus no extra step is required; if not, you may have to create them manually. Alternatively, you may ask Tekton to use a Google Cloud Storage bucket or an AWS Simple Storage Service (Amazon S3) bucket instead. Note that the performance of Tekton may vary depending on the storage option you choose.

>Note
You can check available persistent volumes and storage classes with the commands below:

```bash
kubectl get pv
kubectl get storageclasses
```

These storage options can be configured using ConfigMaps:

## Persistent Volumes

If you would like to configure the size and storage class of the Persistent Volume Tekton requests, update the default config-artifact-pvc configMap. This configMap includes two attributes:

- `size`: the size of the volume
- `storageClassName`: the name of the storage class of the volume

For example, the following example asks Tekton to request a Persistent Volume of 10Gi with the manual storage class when running a workflow:

```bash
kubectl create configmap config-artifact-pvc \
                         --from-literal=size=10Gi \
                         --from-literal=storageClassName=manual \
                         -o yaml -n tekton-pipelines \
                         --dry-run=client | kubectl replace -f -
```

Also, Tekton uses the default service account in your Kubernetes cluster unless otherwise configured; if you would like to override this option, update the default-service-account attribute of the ConfigMap config-defaults:

```bash
kubectl create configmap config-defaults \
                         --from-literal=default-service-account=YOUR-SERVICE-ACCOUNT \
                         -o yaml -n tekton-pipelines \
                         --dry-run=client  | kubectl replace -f -
```

### Adding an S3 bucket as persistent volume

First we need to create the `Secrets` for being able to use S3 as persistent volume

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tekton-storage
  namespace: tekton-pipelines
type: kubernetes.io/opaque
stringData:
  boto-config: |
    [Credentials]
    aws_access_key_id = AWS_ACCESS_KEY_ID
    aws_secret_access_key = AWS_SECRET_ACCESS_KEY
    [s3]
    host = s3.us-east-1.amazonaws.com
    [Boto]
    https_validate_certificates = True
```

Create the config in kubernetes:

`kubectl apply -f secrets.yml -n tekton-pipeline`

Then we add the bucket config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-artifact-bucket
  namespace: tekton-pipelines
data:
  location: s3://tektonbucket
  bucket.service.account.secret.name: tekton-storage
  bucket.service.account.secret.key: boto-config
  bucket.service.account.field.name: BOTO_CONFIG
```

Finally add the bucket as persistent storage:

`kubectl apply -f bucket.yml -n tekton-pipeline`

## CLI

The CLI `tkn` is available on Linux as a .deb package (for Debian, Ubuntu and other deb-based distros) and .rpm package (for Fedora, CentOS, and other rpm-based distros).

```bash
sudo apt update
sudo apt install -y gnupg
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 3EFE0E0A2F2F60AA
echo "deb http://ppa.launchpad.net/tektoncd/cli/ubuntu eoan main"|sudo tee /etc/apt/sources.list.d/tektoncd-ubuntu-cli.list
sudo apt update && sudo apt install -y tektoncd-cli
```

## Sample CI/CD workflow with Tekton

With Tekton, each operation in your CI/CD workflow becomes a `Step`, which is executed with a container image you specify. `Steps` are then organized in `Tasks`, which run as a Kubernetes pod in your cluster. You can further organize `Tasks` into `Pipelines`, which can control the order of execution of several `Tasks`.

To create a `Task`, create a Kubernetes object using the Tekton API with the kind `Task`. The following YAML file specifies a `Task` with one simple Step, which prints a Hello World! message using the official Ubuntu image:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: hello
spec:
  steps:
    - name: hello
      image: ubuntu
      command:
        - echo
      args:
        - "Hello World!"
```

Write the YAML above to a file named `task-hello.yaml`, and apply it to your Kubernetes cluster:

`kubectl apply -f task-hello.yaml -n tekton-pipeline`

To run this task with Tekton, you need to create a `TaskRun`, which is another Kubernetes object used to specify run time information for a Task.

To view this `TaskRun` object you can run the following Tekton CLI (tkn) command:

`tkn task start hello --dry-run -n tekton-pipeline`

After running the command above, the following `TaskRun` definition should be shown:

```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: hello-run-
spec:
  taskRef:
    name: hello
```

To use the `TaskRun` above to start the echo Task, you can either use tkn or kubectl.

with tkn:

`tkn task start hello`

with kubectl:

- use tkn's --dry-run option to save the TaskRun to a file

`tkn task start hello --dry-run > taskRun-hello.yaml`

- create the TaskRun

`kubectl create -f taskRun-hello.yaml -n tekton-pipeline`

Tekton will now start running your `Task`. To see the logs of the last `TaskRun`, run the following tkn command:

`tkn taskrun logs --last -f`

It may take a few moments before your Task completes. When it executes, it should show the following output:

`[hello] Hello World!`

Now let's create a simple pipeline which will run this task; 

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: tutorial-pipeline
spec:
  tasks:
    - name: build-hello-task
      taskRef:
        name: hello
```

Now we need a `PipelineRun` for this `Pipeline` to execute; but before we do that we need to add

- A cluster role for deployment
- A role binding (of course!)

The following YAML defines the cluster role

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: tekton-pipelines
  name: tutorial-role
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "deployments.apps", "replicasets", "pods"]
  verbs: ["*"]
```

Now you tell me to to create the cluster role little kubelets.

Let's add a service account for the `TaskRun`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tutorial-service
  namespace: tekton-pipelines
```

If you'ce reached here you remember most of what we had taught so now we go and create the role binding

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tutorial-role-binding
  namespace: tekton-pipelines
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tutorial-role
subjects:
  - kind: ServiceAccount
    name: tutorial-service
    namespace: tekton-pipelines
```

Apply the role binding to kube cluster.

To run the `Pipeline` first creat a `PipelineRun` as follows:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: tutorial-pipeline-run-1
spec:
  serviceAccountName: tutorial-service
  pipelineRef:
    name: tutorial-pipeline
```

Apply the `PipelineRun`,the `PipelineRun` automatically defines a corresponding `TaskRun` for each `Task` you have defined in your `Pipeline` collects the results of executing each `TaskRun`.

You can monitor the execution of your PipelineRun in realtime as follows:

`tkn pipelinerun logs tutorial-pipeline-run-1 -f`

To view detailed information about your PipelineRun, use the following command:

`tkn pipelinerun describe tutorial-pipeline-run-1`
