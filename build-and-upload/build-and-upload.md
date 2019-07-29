# Building and uploading a docker image inside a KNative cluster

## Initializing steps

### install the build component
```
kubectl apply --filename https://github.com/knative/build/releases/download/v0.7.0/build.yaml
```

## Running the build

### Create a new namespace

  - Create a new configuration file `namepsace.yaml` with the following contents:
    ```yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: build2
    ```
    Command line :
    ```sh
    echo "apiVersion: v1
    kind: Namespace
    metadata:
      name: build2" > namespace.yaml
    ```

### Creating a Kubernetes Sercet for authorizing with docker hub

  - Create an secret configuration file `secret.yaml` to authenticate with your docker hub account
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: basic-user-pass
      namespace: build2
      annotations:
        build.knative.dev/docker-0: https://index.docker.io/v1/  # https://gcr.io for google container registry
    type: kubernetes.io/basic-auth
    stringData:
      username: {username}  # no need for base64 encoding
      password: {password}
    ```
    Replace the `{username}` & `{password}` with your docker hub username and password.

  - Associate a service account with this secret in the configuration file `serviceaccount.yaml`
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: build-bot
      namespace: build2
    secrets:
      - name: basic-user-pass
    ```


### Creating the build configuration file

  - Create a file `build.yaml` with the contents:
    ```yaml
    apiVersion: build.knative.dev/v1alpha1
    kind: Build
    metadata: 
      name: build-helloworld-py
      namespace: build2
      labels:
        expect: succeeded
    spec:
      timeout: 1800s  # timeout specified so that build is not terminated. Default is 10 minutes
      serviceAccountName: build-bot
      source:
        git:
          url: "https://github.com/knative/docs.git"  # git repository where Dockerfile is present
          revision: "release-0.7"  # git branch
        subPath: "docs/serving/samples/hello-world/helloworld-python/"  # path to Dockerfile inside git repo
      steps:
      - name: build-and-push
        image: "gcr.io/kaniko-project/executor:v0.6.0"  # versions above 0.6.0 don't work AFAIK
        args: ["--dockerfile=/workspace/Dockerfile",
              "--destination=docker.io/{username}/py-helloworld:git-source",
              "--skip-tls-verify-pull",
              "--skip-tls-verify"]
    ```
    Replace the `{username}` with your docker hub username.

### Apply the configuration files

  - ```sh
    kubectl apply -f namespace.yaml -f secret.yaml -f serviceaccount.yaml -f build.yaml
    ```

### Monitor the build-and-push status

  - See the pods
    ```sh
    $ kubectl -n=build2 get pods

    NAME                             READY   STATUS     RESTARTS   AGE
    build-helloworld-py-pod-de63c8   0/1     Init:2/3   0          1m
    ```

  - Save the name in a variable
    ```sh
    $ PODNAME=`echo $(kubectl get build hello-build -n=build2 --output=jsonpath={.status.cluster.podName})`
    $ echo $PODNAME

    build-helloworld-py-pod-de63c8
    ```
  - See the status of the image build by kaniko
    ```sh
    $ kubectl -n=build2 logs $PODNAME --container build-step-build-and-push

    INFO[0000] Removing ignored files from build context: [Dockerfile README.md *.pyc *.pyo *.pyd __pycache__] 
    INFO[0000] Downloading base image python:3.7            
    INFO[0111] Taking snapshot of full filesystem...
    ...
    ...
    ```

  - When build is completed (takes around 15 minutes to complete)
    ```
    $ kubectl -n=build2 get pods
    
    NAME                             READY   STATUS      RESTARTS   AGE
    build-helloworld-py-pod-de63c8   0/1     Completed   0          19m
    ```

## Conclusion

  - See the repositories in docker hub to see the new image
  - Local volumes and Google Cloud Services could be used as Dockerfile sources
    - https://knative.dev/docs/build/builds/
  - Instead of a remote docker regsitry, we can host images in a local registry. See https://docs.docker.com/registry/deploying/

## Clean everything

  ```sh
  kubectl delete -f namespace.yaml -f secret.yaml -f serviceaccount.yaml -f build.yaml
  ```