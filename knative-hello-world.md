# Running a KNative Hello-World application

## Requirements

  - Ubuntu 18.04
  - Docker-ce 17.03.3
  - Kubernetes 1.11.0
  - KNative 0.7

**Installation of KNative would not work with other versions AFAIK**


## Steps to install KNative

### Uninstall any other version of docker **if present**

  - To identify what packages you have installed
    ```sh
    dpkg -l | grep -i docker
    ```
  - Switch to `root` user using `sudo -s`
  - Remove the docker packages
    ```
    apt purge -y docker.io docker docker-compose docker-engine
    apt autoremove -y --purge docker.io docker docker-compose docker-engine
    apt remove --purge docker-ce
    ```
  - Remove all the files and directories related to docker
    ```
    rm -rf /var/lib/docker
    rm /etc/apparmor.d/docker
    groupdel docker
    rm -rf /var/run/docker.sock
    rm -rf /usr/local/bin/docker-compose
    rm -rf /etc/docker
    rm -rf ~/.docker
    ```
  - Docker should be completely removed!

### Install Docker-CE 17.03.3

  - Make sure you are looged as `root` user
  - download the deb package
    ```
    wget https://download.docker.com/linux/ubuntu/dists/xenial/pool/stable/amd64/docker-ce_17.03.3~ce-0~ubuntu-xenial_amd64.deb
    ```
  - install
    ```
    dpkg -i docker-ce_17.03.3~ce-0~ubuntu-xenial_amd64.deb
    apt install -f
    ```
  - start the docker service
    ``` 
    systemctl daemon-reload
    systemctl start docker
    ```
  - check the docker installation using `docker run hello-world`

### Uninstall any other version of kubernetes

  - check kubernetes version using `kubeadm version`
  - ensure the version is `1.11.0`
  - Otherwise, follow the belowmentioned steps :
    - `kubeadm reset`
    - `rm -rf ~/.kube`
    - `apt remove --purge kubeadm kubectl kubelet`

### Installing Kubenetes 1.11.0

  - ```
    apt-get update && apt-get install -y apt-transport-https curl
    ```
  - ```
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
    ```
  - ```
    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
    deb http://apt.kubernetes.io/ kubernetes-xenial main
    EOF
    ```
  - `apt install -y kubelet=1.11.0-00 kubeadm=1.11.0-00 kubectl=1.11.0-00 kubernetes-cni=0.6.0-00`
  - Disable swap space
    ```
    swapoff -a
    ```
  - `sed -i "s/cgroup-driver=systemd/cgroup-driver=cgroupfs/g" /etc/systemd/system/kubelet.service.d/10-kubeadm.conf`
  - ```sh
    systemctl daemon-reload
    systemctl restart kubelet
    ```
  
### Initializing Kube Cluster

  - Ensure that you are `root` user.
  - `sysctl net.bridge.bridge-nf-call-iptables=1 `
  - `rm -rf /var/lib/etcd`
  - `kubeadm config images pull`
  - `kubeadm init --pod-network-cidr=10.244.0.0/16`
  - to use kubectl :
    ```sh
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
  - Enable cluster networking
    ```sh
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.10.0/Documentation/kube-flannel.yml
    ```
  - On running `kubectl get pods --all-namespaces`, each pod should show running status after a few minutes.

### Installing KNative

  - `snap install kube-apiserver`
  - enabling the `MutatingAdmissionWebhook` admission contoller
    ```
    kube-apiserver --enable-admission-plugins=NamespaceLifecycle,LimitRanger ...
    ```
  - If it fails, then move forward.

#### Installing Istio(network mesh for KNative)

  - ```sh
    # Download and unpack Istio
    export ISTIO_VERSION=1.1.7
    curl -L https://git.io/getLatestIstio | sh -
    cd istio-${ISTIO_VERSION}
    ```
  - ```
    for i in install/kubernetes/helm/istio-init/files/crd*yaml; do kubectl apply -f $i; done
    ```
  - Create `istio-system` namespace in kubernetes :
    ```sh
    cat <<EOF | kubectl apply -f -
    apiVersion: v1
    kind: Namespace
    metadata:
      name: istio-system
      labels:
        istio-injection: disabled
    EOF
    ```
  - install helm with `snap install helm --classic`
  - ```
    /snap/bin/helm template --namespace=istio-system \
      --set prometheus.enabled=false \
      --set mixer.enabled=false \
      --set mixer.policy.enabled=false \
      --set mixer.telemetry.enabled=false \
      --set pilot.sidecar=false \
      --set pilot.resources.requests.memory=128Mi \
      --set galley.enabled=false \
      --set global.useMCP=false \
      --set security.enabled=false \
      --set global.disablePolicyChecks=true \
      --set sidecarInjectorWebhook.enabled=false \
      --set global.proxy.autoInject=disabled \
      --set global.omitSidecarInjectorConfigMap=true \
      --set gateways.istio-ingressgateway.autoscaleMin=1 \
      --set gateways.istio-ingressgateway.autoscaleMax=1 \
      --set pilot.traceSampling=100 \
      install/kubernetes/helm/istio \
      > ./istio-lean.yaml
    ```
  - ```
    kubectl apply -f istio-lean.yaml
    ```
  - Remove `NoSchedule` taint from node
    ```
    kubectl taint nodes admin1 node-role.kubernetes.io/master:NoSchedule-
    ```
    Replace `admin1` with your node name. To get your Node Name use :
    ```
    kubectl describe nodes | grep Name
    ```
  - To remove any other Taints
    ```
    kubectl describe nodes {NodeName} | grep Taints
    ```
    ```
    kubectl taint nodes {NodeName} {TaintName}-
    ```
  - All the pods should be running after a few minutes
    ```
    kubectl get pods --namespace istio-system
    ```
  - Clean up Istio
    ```sh
    cd ../
    rm -rf istio-${ISTIO_VERSION}
    ```

#### Installing Core KNative

  - ```
    kubectl apply --selector knative.dev/crd-install=true \
    --filename https://github.com/knative/serving/releases/download/v0.7.0/serving.yaml \
    --filename https://github.com/knative/build/releases/download/v0.7.0/build.yaml \
    --filename https://github.com/knative/eventing/releases/download/v0.7.0/release.yaml \
    --filename https://github.com/knative/serving/releases/download/v0.7.0/monitoring.yaml
    ```
  - To complete the install of Knative and its dependencies, run the command again without the `--selector` flag
    ```
    kubectl apply --filename https://github.com/knative/serving/releases/download/v0.7.0/serving.yaml --selector networking.knative.dev/certificate-provider!=cert-manager \
    --filename https://github.com/knative/build/releases/download/v0.7.0/build.yaml \
    --filename https://github.com/knative/eventing/releases/download/v0.7.0/release.yaml \
    --filename https://github.com/knative/serving/releases/download/v0.7.0/monitoring.yaml
    ```
  - Wait for a few minutes and then see the `KNative` pod status. All should be running
    ```sh
    kubectl get pods --namespace knative-serving
    kubectl get pods --namespace knative-build
    kubectl get pods --namespace knative-eventing
    kubectl get pods --namespace knative-monitoring
    ```
    ```sh
    kubectl get pods --all-namespaces
    ```

## Getting first KNative serving application up

We would run a very basic flask app which returns "Hello World"

### Set up docker login for pushing images to the hub

  ```sh
  docker login
  ```
  Provide username and password of your docker hub account when prompted.

  - Create a directory `hello` and `cd` into it.
    ```sh
    mkdir hello
    cd hello
    ```
  - Write a small python script in flask for serving hello-world with `8080` as the serving port
    ```sh
    echo "import os

    from flask import Flask

    app = Flask(__name__)

    @app.route('/')
    def hello_world():
      target = os.environ.get('TARGET', 'World')
      return 'Hello {}!\n'.format(target)

    if __name__ == '__main__':
      app.run(host='0.0.0.0',port=8080)
    " >> hello.py
    ```
    The app can be run using `python3 hello.py` and the output can be seen at `localhost:8080`
  - Dockerize the application. Create a `Dockerfile` in the `hello` directory with the following content.
    ```dockerfile
    FROM python:3.7
    COPY hello.py .
    RUN pip install Flask gunicorn
    EXPOSE 8080
    CMD python hello.py
    ```
    Command line :
    ```sh
    echo "FROM python:3.7
    COPY hello.py .
    RUN pip install Flask gunicorn
    EXPOSE 8080
    CMD python hello.py" >> Dockerfile
    ```
  - Push the docker image to Docker Hub
    ```
    docker build -t rivii/helloworld-python .
    docker push rivii/helloworld-python
    ```
    Replace `rivii` with your Docker Hub username

  - Make a `service.yaml` to deploy on kubernetes kluster and add the following contents to the file
    ```yaml
    apiVersion: serving.knative.dev/v1alpha1
    kind: Service
    metadata:
      name: helloworld-python
      namespace: default
    spec:
      template:
        spec:
          containers:
            - image: docker.io/rivii/helloworld-python
              env:
                - name: TARGET
                  value: "Python Sample v1 | momo"
    ```
    Again, replace `rivii` with your Docker Hub username
  - Deploy it using `kubectl apply --filename service.yaml`
  - To get the Cluster IP address:
    ```sh
    kubectl get svc istio-ingressgateway --namespace istio-system
    ```
    Output:
    ```
    NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
    istio-ingressgateway   LoadBalancer   10.97.172.244   <pending>     15020:31152/TCP,80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:31884/TCP,15030:31113/TCP,15031:30053/TCP,15032:31460/TCP,15443:32640/TCP   3h
    ```
    Note down the Cluster IP. In this example the IP is `10.97.172.244`
  - To find the URL of this `helloworld` service:
    ```
    kubectl get ksvc helloworld-python  --output=custom-columns=NAME:.metadata.name,URL:.status.url
    ```
    Output:
    ```
    NAME                URL
    helloworld-python   http://helloworld-python.default.example.com
    ```
    Note the URL
  - This URL is hosted in a virtual local domain within `10.97.172.244`

  - To get the exepected "Hello World" output:
    ```sh
    curl -H "Host: helloworld-python.default.example.com" http://10.97.172.244
    ```
  - **The output is being served by a KNative service application hosted on a kubernetes cluster.**
  - Finally, to stop the service : 
    ```
    kubectl delete -f service.yaml
    ```
