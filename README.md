[![N|Solid, Docker, and Kubernetes](docs/images/container-banner.jpg)](https://nodesource.com/products/nsolid)

## Overview

This repository is for deploying [N|Solid](https://nodesource.com/products/nsolid) with [Kubernetes](http://kubernetes.io/). It assumes that Kubernetes is already setup for your environment.

![N|Solid, Docker, and Kubernetes](docs/images/kubernetes-cluster.png)

### Table of Contents
- [Installing Kubernetes](#a1)
- [Upgrading](#a1-1)
- [Quickstart](#a2)
    - [Access N|Solid Dashboard](#a3)
    - [Uninstall N|Solid](#a4)
- [Deploy Sample App with N|Solid](#a5)
- [Production Install](#a6)
    - [N|Solid namespace](#a7)
    - [nginx SSL certificates](#a8)
    - [Basic Auth file](#a9)
    - [Secret object for certs](#a10)
    - [Configmap object for settings](#a11)
    - [Define Services](#a12)
    - [GCE persistent disks](#a13)
    - [AWS persistent disks](#a14)
    - [Azure persistent disks](#a29)
    - [Bluemix persistent disks](#a30)
- [Debugging / Troubleshooting](#a15)
    - [Configuring Apps for N|Solid with Kubernetes](#a16)
        - [Building an N|Solid app](#a17)
            - [Docker](#a18)
            - [Kubernetes](#a19)
        - [Accessing your App](#a20)
    - [Accessing N|Solid Kubernetes objects](#a21)
        - [Setting `nsolid` as the default namespace](#a22)
    - [Running `nsolid-cli`](#a23)
    - [minikube](#a24)
        - [Setting ENV for cluster](#a25)
        - [Service Discovery](#a26)
    - [Common Gotchas](#a27)
- [License & Copyright](#a28)

<a name="a1"/>

## Installing kubernetes

* [local with minikube](./docs/install/local.md) - for local development / testing.
* [kubernetes on GKE](./docs/install/GKE.md) - Google Container Engine
* [kubernetes on aws](http://kubernetes.io/docs/getting-started-guides/aws/) - Amazon Web Services
* [kubernetes on GCE](http://kubernetes.io/docs/getting-started-guides/gce/) - Google Compute Engine
* [kubernetes on ACS](http://kubernetes.io/docs/getting-started-guides/azure/) - Microsoft Azure Container Service
* [kubernetes on Bluemix](./docs/install/bluemix-setup.md) - IBM Cloud Container Service

<a name="a1-1"/>

## Upgrading

### local

Existing `nsolid-kubernetes` installs can be upgraded running the following command:

```bash
kubectl apply -f conf/nsolid.quickstart.yml
```

### Cloud

If deployed to a cloud (AWS, Azure, GCP, Bluemix) please make sure to make the necessary adjustments to `conf/nsolid.cloud.yml`

```bash
kubectl apply -f conf/nsolid.cloud.yml
```

<a name="a2"/>

## Quickstart

```bash
./install
```
Notes:
1. Make sure your `kubectl` is pointing to your active cluster.
1. If your cluster is a Bluemix _Lite_ cluster, [make this adjustment](./docs/install/bluemix-lite.md) to conf/nsolid.services.yml before running ./install.

This command will install the N|Solid Console and a secure HTTPS proxy to the `nsolid` namespace.

It can take a little while for Kubernetes to download the N|Solid Docker images.  You can verify
that they are active by running:

```
kubectl --namespace=nsolid get pods
```

When all three pods (console and nginx-secure-proxy) have a status of 'Running', you may continue to access the N|Solid Console.

<a name="a3"/>

### Access N|Solid Dashboard

#### Secure credentials

* Default username: `nsolid`
* Default password: `demo`

#### With `minikube`

```bash
printf "\nhttps://$(minikube ip):$(kubectl get svc nginx-secure-proxy --namespace=nsolid --output='jsonpath={.spec.ports[1].nodePort}')\n"
```

or

#### Cloud Deployment:

```bash
kubectl get svc nginx-secure-proxy --namespace=nsolid
```

Open `EXTERNAL-IP`.  If using Bluemix _Lite_ cluster, get EXTERNAL-IP [this way](./docs/misc/bluemix-external-ip.md).

**NOTE:** You will need to ignore the security warning on the self signed certificate to proceed.

![Welcome Screen](./docs/images/welcome.png)

<a name="a4"/>

### Uninstall N|Solid from Kubernetes cluster

```bash
kubectl delete ns nsolid --cascade
```

<a name="a5"/>

## Deploy Sample App with N|Solid

### Quick Start

```bash
cd sample-app
docker build -t sample-app:v1 .
kubectl create -f sample-app.service.yml
kubectl create -f sample-app.deployment.yml
```

**NOTE:** the container image in `sample-app.deployment.yml` must be set to match your docker image name. E.g. if you are using `minikube` and ran `eval $(minikube docker-env)`, set the image to:

```bash
    spec:
      containers:
        - name: sample-app
          image: sample-app:v1
```

If you are working in a cloud environment, you will need to push the sample-app to a public Docker registry
like [Docker Hub](https://hub.docker.com/), [Quay.io](https://quay.io), the [Azure Container Registry](https://azure.microsoft.com/en-us/services/container-registry/), or the [IBM Bluemix Container Registry](https://console.bluemix.net/docs/services/Registry/registry_images_.html#registry_images_), and update the sample-app Deployment file.


<a name="a6"/>

## Production Install

**NOTE:** Assumes kubectl is configured and pointed at your Kubernetes cluster properly.

<a name="a7"/>

#### Create the namespace `nsolid` to help isolate and manage the N|Solid components.

```
kubectl create -f conf/nsolid.namespace.yml
```

<a name="a8"/>

#### Create nginx SSL certificates

```
openssl req -x509 -nodes -newkey rsa:2048 -keyout conf/certs/nsolid-nginx.key -out conf/certs/nsolid-nginx.crt
```

<a name="a9"/>

#### Create Basic Auth file

```
rm ./conf/nginx/htpasswd
htpasswd -cb ./conf/nginx/htpasswd {username} {password}
```

<a name="a10"/>

#### Create a `secret`  for certs to mount in nginx

```
kubectl create secret generic nginx-tls --from-file=conf/certs --namespace=nsolid
```

<a name="a11"/>

#### Create `configmap` for nginx settings
```
kubectl create configmap nginx-config --from-file=conf/nginx --namespace=nsolid
```

<a name="a12"/>

#### Define the services

```bash
kubectl create -f conf/nsolid.services.yml
```

Note: If your cluster is a Bluemix _Lite_ cluster, [make this adjustment](./docs/install/bluemix-lite.md) to conf/nsolid.services.yml before running `kubectl create`.

#### Create persistent disks

N|Solid components require persistent storage. Kubernetes does not (yet!) automatically handle provisioning of disks consistently across all cloud providers. As such, you will need to manually create the persistent volumes.

<a name="a13"/>

##### On Google Cloud

Make sure the zone matches the zone you brought up your cluster in!

```
gcloud compute disks create --size 10GB nsolid-console
```

<a name="a14"/>

##### On AWS

We need to create our disks and then update the volumeIds in conf/nsolid.persistent.aws.yml.

Make sure the zone matches the zone you brought up your cluster in!

```
aws ec2 create-volume --availability-zone eu-west-1a --size 10 --volume-type gp2
```

<a name="a29"/>

##### On Azure

There's no need to explicitly create a persistent disk, since the Azure Container Service provides a default `StorageClass`, which will dynamically create them as needed (e.g. when a `Pod` includes a `PersistentVolumeClaim`).

<a name="a30"/>

##### On Bluemix

There's no need to explicitly create a persistent disk, since the Bluemix Container Service provides a default `StorageClass`, which will dynamically create them as needed (e.g. when a `Pod` includes a `PersistentVolumeClaim`).


#### Configure Kubernetes to utilize the newly created persistent volumes

##### GCE
```bash
kubectl create -f conf/nsolid.persistent.gce.yml
```

##### AWS
```bash
kubectl create -f conf/nsolid.persistent.aws.yml
```

##### Azure

There's no need to explicitly create a `PersistentVolume` object, since they will be dynamically provisioned by the default `StorageClass`.

##### Bluemix

There's no need to explicitly create a `PersistentVolume` object, since they will be dynamically provisioned by the default `StorageClass`.


#### Deploy N|Solid components

```bash
kubectl create -f conf/nsolid.cloud.yml
```

<a name="a15"/>

## Debugging / Troubleshooting

<a name="a16"/>

### Configuring Apps for N|Solid with Kubernetes

<a name="a17"/>

#### Building an N|Solid app

<a name="a18"/>

##### Docker

Make sure your docker image is build on top of `nodesource/nsolid:carbon-latest`.

```dockerfile
FROM nodesource/nsolid:carbon-latest
```

<a name="a19"/>

##### Kubernetes

When defining your application make sure the following `ENV` are set.

```yaml
  env:
    - name: NSOLID_APPNAME
      value: sample-app
    - name: NSOLID_COMMAND
      value: "console.nsolid:9001"
    - name: NSOLID_DATA
      value: "console.nsolid:9002"
    - name: NSOLID_BULK
      value: "console.nsolid:9003"
```

Optional flags:

```yaml
  env:
    - name: NSOLID_TAGS
      value: "nsolid-carbon,staging"
```

A comma separate list of tags that can be used to filter processes in the N|Solid Console.

<a name="a20"/>

#### Accessing your App

```bash
kubectl get svc {service-name}
```

The `EXTERNAL-IP` will access the application.  
Open `EXTERNAL-IP`.  If using Bluemix _Lite_ cluster, get EXTERNAL-IP [this way](./docs/misc/bluemix-external-ip.md).


<a name="a21"/>

### Accessing N|Solid Kubernetes objects

Make sure you use the `--namespace=nsolid` flag on all `kubectl` commands.

<a name="a22"/>

#### Setting `nsolid` as the default namespace

```bash
kubectl config current-context // outputs current context
kubectl config set-context {$context} --namespace=nsolid // make 'nsolid' the default namespace
kubectl config set-context {$context} --namespace=default // revert to default
```

<a name="a23"/>

### Running `nsolid-cli`

**Verify CLI**:

```bash
kubectl exec {pod-name} -- nsolid-cli --remote=http://console.nsolid:6753 ping
```

See [N|Solid cli docs](https://docs.nodesource.com/nsolid/3.0/docs#using-the-cli) for more info.


<a name="a24"/>

### minikube

Minikube is a bit different then a normal Kubernetes install. The DNS service isn't running so discovering is a bit more involved. IP addresses are not dynamically assigned, instead we must use the host ports the service is mapped to.

<a name="a25"/>

#### Setting ENV for cluster

If your doing a lot of work with docker and minikube it is recommended that you run the following:

```bash
eval $(minikube docker-env)
```

<a name="a26"/>

### Service discovery

Get the kubernetes cluster ip address:

```bash
minikube ip
```

To get the service port:

```bash
kubectl get svc {$service-name} --output='jsonpath={.spec.ports[0].nodePort}'
```

**Note:** If your service exposes multiple ports you may want to examine with `--output='json'` instead.


<a name="a27"/>

### Common Gotchas

If you get the following message when trying to run `docker build` or communicating with the Kubernetes API.

```bash
Error response from daemon: client is newer than server (client API version: 1.24, server API version: 1.23)
```

Export the `DOCKER_API_VERSION` to match the server API version.

```bash
export DOCKER_API_VERSION=1.23
```

<a name="a28" />

## License & Copyright

**nsolid-kubernetes** is Copyright (c) 2018 NodeSource and licensed under the MIT license. All rights not explicitly granted in the MIT license are reserved. See the included [LICENSE.md](LICENSE.md) file for more details.
