# What is kubernetes? (Issue 1)

"Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications."
- Use the [kubernetes documentation](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/) to learn about kubernetes clusters.
- Make sure of understanding nodes, pods, deployments and services concepts.

# Creating a kubernetes single node cluster running a flask application (Issue 2)

This is a tutorial of how to setup a fully working Kubernetes single node cluster. In this tutorial we will use the k3s kubernetes lightweight distribution because of its ability to easily create single nodes clusters as well as create expandable clusters.
- You can also choose to use others k8s distributions like:
  - [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
  - [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
  - others
- Read this [discussion](https://www.reddit.com/r/kubernetes/comments/be0415/k3s_minikube_or_microk8s/) to help choosing your distribution.
## 1 - Install k3s with kubeconfig mode 664

Run the following commands to install a single node k3s cluster on your local machine as a systemd service.

``` console
$ curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--write-kubeconfig-mode 664" sh -
```

- This command will install the latest k3s distribution with some usefull sevices like kubectl and let you acess the kubeconfig file whithout needing to type sudo every time. 

Run those to confirm the installation.

``` console
$ k3s --version
$ kubectl version
```

- If you still get perminssions errors, run :

``` console
$ sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

## 2 - Starting a pod
Create a deployment using the [arkaisho/hello_flask](https://hub.docker.com/repository/docker/arkaisho/hello_flask) image from dockerhub.

``` console
$ kubectl run flask --image=arkaisho/hello_flask --port=5000
```

- The image used is a simple flask application exposed on the 5000 port, when you specificate the --port=5000 flag you make the cluster publish that port for outer acess.

## 3 - Confirm the creation of the flask pod
Use this command to watch the alterations happening in the cluster pods until the 'flask' pod get running (it should take about 4 minutes depending on your connection).

``` console
$ kubectl get pod -w
```
Use ctrl+c to get out of the command.

## 4 - Make the application accessible
Run this command to create a service exposing the pod as a cluster service.

``` console
$ kubectl expose deployment flask --type=LoadBalancer --port=5000 --target-port=5000
```

- The --port=5000 flag make the service get open on the 5000 port instead of a random one.
- The --target-port=5000 flag say to the service to foward requests through the 5000 container port.

## 5 - Confirm service creation
Run this command to confirm that you have a service pointing to your application.

``` console
$ kubectl get service
```
## 6 - Acess your service
Now you can just acess your [site](http://localhost:5000) on your local machine.

## 7 - Delete your application
To delete your appliaction run:
``` console
$ kubectl delete service flask
$ kubectl delete deployment flask
```
# Working with namespaces (Issue 3)

"Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces."
Read about kubernetes [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) to know how they works and when to use they.

## 1 - Create your namespace

Use the following command to create a namespace called "dev"

``` console
$ kubectl create namespace dev
```

## 2 - How to use it 

To work with "kubectl" commands in a specific namespace you just have to insert "-n namespace-name" where "namespace-name" is the name of the namespace you want to work with:

``` console
$ kubectl apply -f notebookpod.yaml -n dev
pod/notebook created
$ kubectl get pod
No resources found in default namespace.
$ kubectl get pod -n dev
NAME       READY   STATUS    RESTARTS   AGE
notebook   1/1     Running   0          8s
$ kubectl delete pod notebook -n dev
pod "notebook" deleted
```
- Here we've created a pod within the namespace "dev" and then tried to see this pod without the "-n dev" flag, notice theres is no resources found in default namespace.
- When we use the "-n dev" flag we can see the pod and make other commands like delete it.

> You can also use [kubens](https://github.com/ahmetb/kubectx) to work with namespace easily.

# Creating a jupyter notebook server inside the cluster (Issue 3)

It is useful to have a jupyter server stealing in your cluster, you can use it as a gateway to manipulate your cluster.

## 1 - Create the notebook deployment

Different from the last session we will use the command "kubectl apply" to put a deployment into your cluster. The files used in this session are inside this repository at "issue3" folder. We will use the image [arkaisho/minimal-notebook](https://hub.docker.com/repository/docker/arkaisho/minimal-notebook) that is a [jupyter/minimal-notebook](https://hub.docker.com/r/jupyter/minimal-notebook) with some configurations:
 - the notebook is exposed on the default port 8888
 - the container have sudo permissions granted
 - the container have the python pykube-ng package already installed (we will use it in the next sessions)
 - the container home folder is open to mount volumes into it (this will be useful in #issue 5)
 - the notebook server has no authentication, so you can get into the HomePage easily

To create the deployment get inside the issue3 folder and run:

``` console
$ kubectl apply -f deployment.yaml
```
You can check the status of the automatically created pod using:

``` console
$ kubectl get pod
```

lets take a look at the deployment.yaml file to understand what we did.
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notebook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: notebook
  template:
    metadata:
      labels:
        app: notebook
    spec:
      containers:
      - image: arkaisho/minimal-notebook
        name: notebook
        ports:
        - containerPort: 8888
```
the fields in this file describe something like we did in Issue 2.
 * apiVersion: the api that we are using to manipulate the cluster.
 * kind: the kind of object tha the file describes (deployment, service, pod).
 * metadata: data about the created object.
 * spec: object configuration.
     * replicas: number of pods generateds by the deployment.
     * selector: [labels and selectors can be used to organize and to select subsets of objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/).
     * template: the pod template to be replicated.
        * container: configuration of the container inside the pod.

## 2 - Create the service to acess the notebook

Use the command kubectl expose to create the service:

``` console
$ kubectl apply -f service.yaml
```

Use the kubectl get svc (service abbreviation) to get service status

``` console
$ kubectl get service
```
like the deployment, the service.yaml also describes the service created.
``` yaml
apiVersion: v1
kind: Service
metadata:
  name: notebook
spec:
  ports:
  - port: 8888
    protocol: TCP
    targetPort: 8888
  selector:
    app: notebook
  type: LoadBalancer
```
the fields in this file also describe something like we did in Issue 2.
 * spec: configuration of the service.
    * ports: set of port fowards to acess the pod.
    * selector: used to match the deployment and pod labels.
    * type: type of acess to the service. Read about [here](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types).

## 3 - Acess the notebook

Now you can just acess your [jupyter notebook server](http://localhost:8888) on your local machine.

# Using pykube-ng to control the cluster from within the jupyter notebook pod (Issue 4)

[Pykube-ng](https://pypi.org/project/pykube-ng/) is a lightweight python client library for kubernetes. It is a fork of [kelporoject/pykube](https://github.com/kelproject/pykube) which is no longer maintained.

## 1 - Preparing to use the pykube inside the cluster

In order to have acess to kubernetes API inside the cluster we have some things to do:

- Put the config file of your cluster inside the notebook container
- Copy the tutorial files into the container
- Get inside the pod to change the configuration file

### 1.1 - Getting the config file inside the cluster
First you need to get the notebook pod name created by the deployment. You can do it by using the "kubectl get pod" command. In my case, the pod name is "notebook-65bcf48946-ddqht".

``` console
$ kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
notebook-65bcf48946-ddqht   1/1     Running   0          108s
svclb-notebook-6pw7d        1/1     Running   0          21s
```

Because i'm using the k3s, my config file is in "/etc/rancher/k3s/k3s.yaml". You can copy it into your cluster using the kubectl cp command and the pod name.

```console
$ kubectl cp /etc/rancher/k3s/k3s.yaml notebook-65bcf48946-ddqht:k3s.yaml
```
- When we use "notebook-65bcf48946-ddqht:config" as the destination for the file we are saving it inside the pod main folder with the name "config" 

You can use the same command to copy the "pku.py", "Issue5.ipynb" and the "Issue4.ipynb" files into the main folder. But if your prefer you can clone this repository inside the cluster to get the files in there.

```console
$ kubectl cp pku.py <your-pod-name>:pku.py
$ kubectl cp Issue4.ipynb <your-pod-name>:Issue4.ipynb
$ kubectl cp Issue5.ipynb <your-pod-name>:Issue5.ipynb
```

### 1.2 - Get inside the running pod

The config file have a server field that point to the kubernetes api server of your cluster. By now, you must have something like:
```console
server: https://127.0.0.1:6443
```
It works outside of your single-node cluster, but the API ip address inside the cluster is different. Lets fix it. Use the command get svc to see the api address inside the cluster.

```console
$ kubectl get service
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.43.0.1      <none>        443/TCP          12d
notebook     LoadBalancer   10.43.180.77   192.168.1.8   8888:31154/TCP   12m
```
> The "kubernetes" service is the service containnig the api. The CLusterIp is the ip address of the api inside the cluster.

To change the address in that file (that is inside the cluster) you need to get a bash inside your pod, you can do it using the kubectl exec command:

```console
$ kubectl exec -ti <your-pod-name> -- /bin/bash
```
Use the "ls" command to see that your config file is there. Use Nano to edit the file and change the server field to the api address with the port 443.

```console
$ nano config
```
leave the server field like this:
```console
server: https://<your-kubernetes-api-ip-address>:443 
```

Now you can import pykube and operator to query requests to the kubernetes api. Open your notebook and paste this code to test:

```python
import pykube
import operator

api = pykube.HTTPClient(pykube.KubeConfig.from_file("config"))
pods = pykube.Pod.objects(api)

for pod in pods:
  print(pod.name)
```

Now open the Issue4.ipynb and test some freatures of the pykube-ng.

# Next sessions

As we will use pykube in the next sessions, they will be on their respective notebooks.

# Usefull commands

## kill all cluster processes
```console
$ k3s-killall.sh
```
## see cluster token
```console
$ sudo cat /var/lib/rancher/k3s/server/node-token
```
## see cluster config file
```console
$ sudo cat /etc/rancher/k3s/k3s.yaml
```
## disable cluster auto start
```console
$ systemctl disable k3s.service
```
## stop running cluster
```console
$ systemctl stop k3s.service
```
## start stopped cluster
```console
$ systemctl restart k3s.service
```