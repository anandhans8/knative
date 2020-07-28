# Docker Hub Build

## Register secrets for Docker Hub

We need to first register a secret in Kubernetes for authentication with Docker Hub.

Create a [docker-secret.yaml](../build/docker-secret.yaml) file for `Secret` manifest, which is used to store your Docker Hub credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-user-pass
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  # Use 'echo -n "username" | base64' to generate this string
  username: BASE64_ENCODED_USERNAME
  # Use 'echo -n "password" | base64' to generate this string
  password: BASE64_ENCODED_PASSWORD
```

Make sure to replace `BASE64_ENCODED_USERNAME` and `BASE64_ENCODED_PASSWORD` with your Base64 encoded DockerHub username and password.

Create a [service-account.yaml](../build/service-account.yaml) for `Service Account` used to link the build process to the secret:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: docker-build-bot
secrets:
  - name: docker-user-pass
```

Apply the `Secret` and `Service Account`:

```bash
kubectl apply -f docker-secret.yaml
secret "docker-user-pass" created

kubectl apply -f service-account.yaml
serviceaccount "docker-build-bot" created
```

## Design the Task

[Kaniko](https://github.com/GoogleContainerTools/kaniko) is a tool to build container images from a Dockerfile, inside a container in Kubernetes cluster. The advantage is that Kaniko doesn't depend on a Docker daemon. Let's create a generic Task to use Kaniko.

Create a [task-build-docker-images-from-git-source.yaml](../build/task-build-docker-images-from-git-source.yaml) file:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  inputs:
    resources:
      - name: docker-source
        type: git
    params:
      - name: pathToDockerFile
        type: string
        description: The path to the dockerfile to build
        default: /workspace/docker-source/Dockerfile
      - name: pathToContext
        type: string
        description:
          The build context used by Kaniko
          (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
        default: /workspace/docker-source
  outputs:
    resources:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.9.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/builder/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(inputs.params.pathToDockerFile)
        - --destination=$(outputs.resources.builtImage.url)
        - --context=$(inputs.params.pathToContext)
```

This defines a generic Task to use Kaniko to download the source code in the 'workspace' directory and then use Kaniko to build and push an image to Docker Hub.

Create the task:

```bash
kubectl apply -f task-build-docker-images-from-git-source.yaml

task.tekton.dev/build-docker-image-from-git-source created
```

## Design the TaskRun

Create a [taskrun-build-helloworld-docker.yaml](../build/taskrun-build-helloworld-docker.yaml) file:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-knative-tutorial
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      # Point to Git url
      value: https://github.com/anandhans8/knative
---
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: image-docker-knative-tutorial
spec:
  type: image
  params:
    - name: url
      # Replace {username} with your actual DockerHub
      value: docker.io/{username}/helloworld:tekton
---
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: build-helloworld-docker
spec:
  serviceAccountName: docker-build-bot
  taskRef:
    name: build-docker-image-from-git-source
  inputs:
    resources:
      - name: docker-source
        resourceRef:
          name: git-knative-tutorial
    params:
      - name: pathToDockerFile
        value: Dockerfile
      - name: pathToContext
        # Point to Dockerfile
        value: /workspace/docker-source/serving/helloworld-csharp 
  outputs:
    resources:
      - name: builtImage
        resourceRef:
          name: image-docker-knative-tutorial
```

## Run and watch the TaskRun

You can start the build with:

```bash
kubectl apply -f taskrun-build-helloworld-docker.yaml

pipelineresource.tekton.dev/git-knative-tutorial unchanged
pipelineresource.tekton.dev/image-docker-knative-tutorial created
taskrun.tekton.dev/build-helloworld-docker created
```

Check that all the Tekton artifacts are created:

```bash
kubectl get tekton-pipelines
```

Soon after, you'll see a pod created for the build:

```bash
kubectl get pods

NAME                                     READY     STATUS
build-helloworld-docker-pod-a1d405        0/4       Init:2/3
```

You can see the progress of the build with:

```bash
kubectl logs --follow --container=step-build-and-push <podid>
```

When the build is finished, you'll see the pod in `Completed` state:

```bash
kubectl get pods

NAME                                     READY     STATUS
build-helloworld-docker-pod-a1d405        0/4       Completed
```

At this point, you should see the image pushed to Docker Hub:

![Docker Hub](./docker-hub.png)
