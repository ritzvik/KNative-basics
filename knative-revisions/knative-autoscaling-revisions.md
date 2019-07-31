# Autoscaling & Revisions with KNative

### Why do we need Revisions?

  - In a production environment, software rollout can be in stages, where one might want a limited number of users to experience the new version.

### Let's create a new namespace

  ```

  $ cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Namespace
  metadata:
    name: revs
  EOF
  
  ```

### Create two `.py` Flask apps for different revisions

  - File 1:
    ```sh
    $ echo "import os

    from flask import Flask

    app = Flask(__name__)

    @app.route('/')
    def hello_world():
      target = os.environ.get('TARGET', 'World')
      return 'Hello {} : v1 !\n'.format(target)

    if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8080)
    " >> hello1.py
    ```
  
  - File 2:
    ```sh
    $ echo "import os

    from flask import Flask

    app = Flask(__name__)

    @app.route('/')
    def hello_world():
      target = os.environ.get('TARGET', 'World')
      return 'Hello {} : v2 !\n'.format(target)

    if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8080)
    " >> hello2.py
    ```
  
  - Dockerfile 1
    ```sh
    $ echo "FROM python:3.7
    COPY hello1.py .
    RUN pip install Flask gunicorn
    EXPOSE 8080
    CMD python hello1.py" >> Dockerfile_1
    ```

  - Dockerfile 2
    ```sh
    $ echo "FROM python:3.7
    COPY hello2.py .
    RUN pip install Flask gunicorn
    EXPOSE 8080
    CMD python hello2.py" >> Dockerfile_2
    ```

  - Build and push the images to local registry
    ```sh
    docker build -t rivii/helloworld-python:v1 . -f Dockerfile_1
    docker push rivii/helloworld-python:v1
    docker build -t rivii/helloworld-python:v2 . -f Dockerfile_2
    docker push rivii/helloworld-python:v2
    ```
    replace `rivii` by your DockerHub username

  - Now all the images are in remote registry

### Create the first service

  - create `service.yaml` with the following contents:
    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
      name: helloworld-python
      namespace: revs
    spec:
      runLatest:
        configuration:
          revisionTemplate:
            metadata:
              # autoscaling.kantive.dev/target: "3"
              # autoscaling.knative.dev/minScale: "1"
            spec:
              container:
                image: docker.io/rivii/helloworld-python:v1
    ```

    - Some explanation of the terms here:
      - `autoscaling.kantive.dev/target: "3"` means that one pod would handle at-max 3 requests at a time. This would be beneficial when processing is heavy and you don't want to route large number of requests to one pod!
      - `autoscaling.knative.dev/minScale: "1"` means that there would be atleast one pod running the whole time. This is benificial when the traffic drops to 0 sometimes, and you don't want latency when there is a request.
      - Uncomment these two lines if required.

### Get response from the service
  - To get the Cluster IP address:
    ```sh
    kubectl get svc istio-ingressgateway --namespace istio-system
    ```
  - In my case it's `10.108.97.143`
  - ```sh
    $ curl -H "Host: helloworld-python.revs.example.com" http://10.108.97.143

    Hello World : v1 !
    ```

### Edit the configuration file to deploy a new revision

  - edit the `service.yaml` file with the following contents:
    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
      name: helloworld-python
      namespace: revs
    spec:
      runLatest:
        configuration:
          revisionTemplate:
            metadata:
              # autoscaling.kantive.dev/target: "3"
              # autoscaling.knative.dev/minScale: "1"
            spec:
              container:
                image: docker.io/rivii/helloworld-python:v2
    ```

  - apply
    ```
    kubectl apply -f service.yaml
    ```
  - after a few seconds, new revision would be deployed
    ```sh
    $ curl -H "Host: helloworld-python.revs.example.com" http://10.108.97.143

    Hello World : v2 !
    ```

  - get all the revisions deployed
    ```
    $ kubectl get revisions -n=revs

    NAME                      SERVICE NAME              GENERATION   READY   REASON
    helloworld-python-nhhnw   helloworld-python-nhhnw   2            True    
    helloworld-python-th2bq   helloworld-python-th2bq   1            True
    ```
  - my first generation revision is :
    ```
    helloworld-python-th2bq
    ```

### Splitting the traffic between revisions

  - Suppose I want to redirect 50% of the traffic to revision 1
  - Change the `service.yaml` file and apply.
    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
      name: helloworld-python
      namespace: revs
    spec:
      release:
        revisions:
          - helloworld-python-th2bq
          - helloworld-python-nhhnw
        rolloutPercent: 50
        configuration:
          revisionTemplate:
            metadata:
              # autoscaling.kantive.dev/target: "3"
              # autoscaling.knative.dev/minScale: "1"
            spec:
              container:
                image: docker.io/rivii/helloworld-python:v2
    ```
  - To see the traffic split, `curl` the service 20 times:
    ```sh
    $ for ((i=1;i<=20;i++)); do   curl -H "Host: helloworld-python.revs.example.com" http://10.108.97.143; done

    Hello World : v1 !
    Hello World : v2 !
    Hello World : v1 !
    Hello World : v2 !
    Hello World : v1 !
    Hello World : v2 !
    Hello World : v1 !
    Hello World : v2 !
    Hello World : v2 !
    Hello World : v2 !
    Hello World : v2 !
    Hello World : v2 !
    Hello World : v1 !
    Hello World : v1 !
    Hello World : v2 !
    Hello World : v1 !
    Hello World : v1 !
    Hello World : v1 !
    Hello World : v2 !
    Hello World : v2 !

    ```
  - The traffic is split approximately 50%
  