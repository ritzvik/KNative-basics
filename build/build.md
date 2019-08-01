# Creating a simple KNative build
reference : *https://knative.dev/docs/build/creating-builds/*

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
      name: simplebuild
    ```
    Command line :
    ```sh
    echo "apiVersion: v1
    kind: Namespace
    metadata:
      name: simplebuild" > namespace.yaml
    ```
  - Apply the config file:
    ```sh
    kubectl apply -f namespace.yaml
    ```

### Creating the build

  - Create the build configuration file `build.yaml` with the above namespace:
    ```yaml
    apiVersion: build.knative.dev/v1alpha1
    kind: Build
    metadata:
      name: hello-build
      namespace: simplebuild
    spec:
      steps:
        - name: hello
          image: busybox
          args: ["echo", "hello", "build"]
    ```
    Command line:
    ```sh
    echo 'apiVersion: build.knative.dev/v1alpha1
    kind: Build
    metadata:
      name: hello-build
      namespace: simplebuild
    spec:
      steps:
        - name: hello
          image: busybox
          args: ["echo", "hello", "build"]
    ' > build.yaml
    ```
    Apply the config file
    ```sh
    kubectl apply -f build.yaml
    ```

### Getting the output of build

  - get the pod name
  - To see the pod name:
    ```sh
    $ kubectl -n=simplebuild get pods

    NAME                     READY   STATUS      RESTARTS   AGE
    hello-build-pod-919203   0/1     Completed   0          12m
    ```
    Also:
    ```sh
    $ HELLOPODNAME=`echo $(kubectl get build hello-build -n=simplebuild --output=jsonpath={.status.cluster.podName})`
    $ echo $HELLOPODNAME

    hello-build-pod-919203
    ```
  - To see the output:
    ```sh
    $ kubectl -n=simplebuild logs $HELLOPODNAME --container=build-step-hello

    hello build
    ```

### Clean the pods and namespaces:

  ```
  kubectl delete -f build.yaml -f namespace.yaml
  ```
