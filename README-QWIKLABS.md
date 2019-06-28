# Migrating to Containers

Containers are quickly becoming an industry standard for deployment of software applications. The business and technological advantages of containerizing workloads are driving many teams towards moving their applications to containers. This demo provides a basic walkthrough of migrating a stateless application from running on a VM to running on [Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/). It demonstrates the lifecycle of an application transitioning from a typical VM/OS-based deployment to a specialized os for containers to a platform for containers better known as [GKE](https://cloud.google.com/kubernetes-engine/).

## Table of Contents
<!-- TOC -->
  * [Table of Contents](#table-of-contents)
  * [Introduction](#introduction)
  * [Architecture](#architecture)
  * [Initial Setup](#initial-setup)
     * [Configure gcloud](#configure-gcloud)
     * [Get The Code](#get-the-code)
     * [Tools](#tools)
        * [Install Cloud SDK](#install-cloud-sdk)
        * [Install kubectl CLI](#install-kubectl-cli)
        * [Install Terraform](#install-terraform)
     * [Authenticate gcloud](#authenticate-gcloud)
  * [Deployment](#deployment)
  * [Exploring Prime Flask Environments ](#exploring-prime-flask-environments)
  * [Validation](#validation)
  * [Load Testing](#load-testing)
  * [Tear Down](#tear-down)
  * [More Info](#more-info)
  * [Troubleshooting](#troubleshooting)
<!-- TOC -->

## Introduction

There are numerous advantages to using containers to deploy applications. Among these are:

1. _Isolated_ - Applications have their own libraries; no conflicts will arise from different libraries in other applications.

1. _Limited (limits on CPU/memory)_ - Applications may not hog resources from other applications.

1. _Portable_ - The container contains everything it needs and is not tied to an OS or Cloud provider.

1. _Lightweight_ - The kernel is shared, making it much smaller and faster than a full OS image.

***What you'll learn***
This project demonstrates migrating a simple Python application named [Prime-flask](container/prime-flask-server.py) to:

1.  A [virtual machine (Debian VM)](https://cloud.google.com/compute/docs/instances/) where [Prime-flask](container/prime-flask-server.py) is deployed as the only application, much like a traditional application is run in an on-premises datacenter

1.  A containerized version of [Prime-flask](container/prime-flask-server.py) is deployed on [Container-Optimized OS (COS)](https://cloud.google.com/container-optimized-os/)

1.  A [Kubernetes](https://kubernetes.io/) deployment where `Prime-flask` is exposed via a load balancer and deployed in [Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)

After the deployment you'll run a load test against the final deployment and scale it to accommodate the load.

The python app [Prime-flask](container/prime-flask-server.py) has instructions for creating container in [this folder](container).

## Architecture

**Configuration 1:** [Virtual machine](https://cloud.google.com/compute/docs/instances/) running Debian, app deployed directly to the host OS, no containers

![screenshot](./images/Debian-deployment.png)

**Configuration 2:** Virtual machine running [Container-Optimized OS](https://cloud.google.com/container-optimized-os/), app deployed into a container

![screenshot](./images/cos-deployment.png)

**Configuration 3:** [Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine/) platform, many machines running many containers

![screenshot](./images/gke-deployment.png)

A simple Python [Flask](http://flask.pocoo.org/) web application (`Prime-flask`) was created for this demonstration which contains two endpoints:

`http://<ip>:8080/factorial/` and

`http://<ip>:8080/prime/`

Examples of use would look like:

```console
curl http://35.227.149.80:8080/prime/10
The sum of all primes less than 10 is 17

curl http://35.227.149.80:8080/factorial/10
The factorial of 10 is 3628800
```

Also included is a utility to validate a successful deployment.

## Initial Setup

### Configure gcloud

When using Cloud Shell execute the following command in order to setup gcloud cli.

```console
gcloud init
```


### Get The Code

* [Fork the repo](https://help.github.com/articles/fork-a-repo/)
* [Clone your fork](https://help.github.com/articles/cloning-a-repository/)

## Deployment

The infrastructure required by this project can be deployed by executing:
```console
make create
```

This will call script [create.sh](scripts/create.sh) which will perform following tasks:
1.  Package the deployable `Prime-flask` application, making it ready to be copied to [Google Cloud Storage](https://cloud.google.com/storage/).
1.  Create the container image via [Google Cloud Build](https://cloud.google.com/cloud-build/) and push it to the private [Container Registry (GCR)](https://cloud.google.com/container-registry/) for your project.
1.  Generate an appropriate configuration for [Terraform](https://www.terraform.io).
1.  Execute Terraform which creates the scenarios we outlined above.

Terraform creating single VM, COS VM, and GKE cluster:

![screenshot](./images/setup.png)

Terraform outputs showing prime and factorial endpoints for Debian VM and COS system:

![screenshot](./images/setup-2.png)

Kubernetes Cluster and [Prime-flask](container/prime-flask-server.py) service are up:

![screenshot](./images/setup-success.png)

## Exploring Prime Flask Environments

We have now setup three different environments that our [Prime-flask](container/prime-flask-server.py) app could traverse as it is making its way to becoming a container app living on a single virtual machine to a [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) running on a container orchestration platform like [Kubernetes](https://kubernetes.io/).

At this point it would benefit you to explore the systems.

Jump onto the Debian [virtual machine](https://cloud.google.com/compute/docs/instances/), `vm-webserver`, that has application running on host OS. In this environment there is no isolation, and portability is less efficient. In a sense the app running on the system has access to all the system and depending on other factors may not have automatic recovery of application if it fails. Scaling up this application may require to spin up more virtual machines and most likely will not be best use of resources.
```
gcloud compute ssh vm-webserver --zone us-west1-c
```

List all processes:
```bash
ps aux
```
```bash
root       882  0.0  1.1  92824  6716 ?        Ss   18:41   0:00 sshd: user [priv]
user       888  0.0  0.6  92824  4052 ?        S    18:41   0:00 sshd: user@pts/0
user       889  0.0  0.6  19916  3880 pts/0    Ss   18:41   0:00 -bash
user       895  0.0  0.5  38304  3176 pts/0    R+   18:41   0:00 ps aux
apprunn+  7938  0.0  3.3  48840 20328 ?        Ss   Mar19   1:06 /usr/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
apprunn+ 21662  0.0  3.9  69868 24032 ?        S    Mar20   0:05 /usr/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
```

Jump onto the [Container-Optimized OS (COS)](https://cloud.google.com/container-optimized-os/) machine, `cos-vm`, where we have docker running the container. COS is an optimized operating system with small OS footprint, which is part of what makes it secure to run container workloads. It has cloud-init and has Docker runt time preinstalled. This system on its own could be great to run several containers that did not need to be run on a platform that provided higher levels of reliability.

```bash
gcloud compute ssh cos-vm --zone us-west1-c
```

We can also run `ps aux` on the host and see the prime-flask running, but notice docker and container references:
```bash
root         626  0.0  5.7 496812 34824 ?        Ssl  Mar19   0:14 /usr/bin/docker run --rm --name=flaskservice -p 8080:8080 gcr.io/migration-to-containers/prime-flask:1.0.2
root         719  0.0  0.5 305016  3276 ?        Sl   Mar19   0:00 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 8080 -container-ip 172.17.0.2 -container-port 8080
root         724  0.0  0.8 614804  5104 ?        Sl   Mar19   0:09 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/mo
chronos      741  0.0  0.0    204     4 ?        Ss   Mar19   0:00 /usr/bin/dumb-init /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
chronos      774  0.0  3.2  21324 19480 ?        Ss   Mar19   1:25 /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
chronos    14376  0.0  4.0  29700 24452 ?        S    Mar20   0:05 /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
```

Also notice if we try to list the python path it does not exist:
```bash
ls /usr/local/bin/python
ls: cannot access '/usr/local/bin/python': No such file or directory
```

Docker list containers
```bash
docker ps
```
```bash
CONTAINER ID        IMAGE                                                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
d147963ec3ca        gcr.io/migration-to-containers/prime-flask:1.0.2   "/usr/bin/dumb-init …"   39 hours ago        Up 39 hours         0.0.0.0:8080->8080/tcp   flaskservice
```

Now we can exec a command to see running process on the container:
```bash
docker exec -it $(docker ps |awk '/prime-flask/ {print $1}') ps aux
```
```bash
PID   USER     TIME  COMMAND
    1 apprunne  0:00 /usr/bin/dumb-init /usr/local/bin/gunicorn --bind 0.0.0.0:
    6 apprunne  1:25 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn -
   17 apprunne  0:05 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn -
   29 apprunne  0:00 ps aux
```

Jump on to [Kubernetes](https://kubernetes.io/). In this environment we can run hundreds or thousands of [pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/) that are groupings of containers. Kubernetes is the defacto standard for deploying containers these days. It offers high productivity, reliability, and scalability. Kubernetes makes sure your containers have a home, and if container happens to fail, it will respawn it again. You can have many machines making up the cluster and in so doing you can spread it across different zones ensuring availability, and resilience to potential issues.

Get cluster configuration:
```bash
gcloud container clusters get-credentials prime-server-cluster
```

Get pods running in the default namespace:
```bash
kubectl get pods
```
```bash
NAME                            READY   STATUS    RESTARTS   AGE
prime-server-6b94cdfc8b-dfckf   1/1     Running   0          2d5h
```

See what is running on the pod:
```bash
kubectl exec $(kubectl get pods -lapp=prime-server -ojsonpath='{.items[].metadata.name}')  -- ps aux
```
```bash
PID   USER     TIME  COMMAND
    1 apprunne  0:00 /usr/bin/dumb-init /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
    6 apprunne  1:16 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
    8 apprunne  2:52 {gunicorn} /usr/local/bin/python /usr/local/bin/gunicorn --bind 0.0.0.0:8080 prime-flask-server
   19 apprunne  0:00 ps aux
```

As you can see from the last example, python application is now running in a container. The application can't access anything on the host. The container is isolated. It runs in a linux namespace and can't (by default) access files, the network, or other resources running on the VM, in containers or otherwise.

## Validation

Now that the application is deployed, we can validate these three deployments by executing:

```console
make validate
```

A successful output will look like this:
```console
Validating Debian VM Webapp...
Testing endpoint http://35.227.149.80:8080
Endpoint http://35.227.149.80:8080 is responding.
**** http://35.227.149.80:8080/prime/10
The sum of all primes less than 10 is 17
The factorial of 10 is 3628800

Validating Container OS Webapp...
Testing endpoint http://35.230.123.231:8080
Endpoint http://35.230.123.231:8080 is responding.
**** http://35.230.123.231:8080/prime/10
The sum of all primes less than 10 is 17
The factorial of 10 is 3628800

Validating Kubernetes Webapp...
Testing endpoint http://35.190.89.136
Endpoint http://35.190.89.136 is responding.
**** http://35.190.89.136/prime/10
The sum of all primes less than 10 is 17
The factorial of 10 is 3628800
```

Of course, the IP addresses will likely differ for your deployment.

## Load Testing

In a new console window, execute the following, replacing `[IP_ADDRESS]` with the IP address and port from your validation output from the previous step. Note that the Kubernetes deployment runs on port `80`, while the other two deployments run on port `8080`:
```console
ab -c 120 -t 60  http://<IP_ADDRESS>/prime/10000
```
ApacheBench (`ab`) will execute 120 concurrent requests against the provided endpoint for 1 minute. The demo application's replica is insufficiently sized to handle this volume of requests.

This can be confirmed by reviewing the output from the `ab` command. A `Failed requests` value of greater than 0 means that the server couldn't respond successfully to this load:

![screenshot](./images/ab_load-test-1.png)

One way to ensure that your system has capacity to handle this type of traffic is by scaling up. In this case, we would want to scale our service horizontally.

In our Debian and COS architectures, horizontal scaling would include:

1. Creating a load balancer.
1. Spinning up additional instances.
1. Registering them with the load balancer.

This is an involved process and is out of scope for this demonstration.

For the third (Kubernetes) deployment the process is far easier:

```console
kubectl scale --replicas 3 deployment/prime-server
```

After allowing 30 seconds for the replicas to initialize, re-run the load test:

```console
ab -c 120 -t 60  http://<IP_ADDRESS>/prime/10000
```
Notice how the `Failed requests` is now 0. This means that all of the 10,000+ requests were successfully answered by the server:

![screenshot](./images/ab_load-test-2.png)

## Tear Down

When you are finished with this example you will want to clean up the resources that were created so that you avoid accruing charges:

```
$ make teardown
```

It will run `terraform destroy` which will destroy all of the resources created for this demonstration.

![screenshot](./images/tear-down.png)

## More Info
For additional information see: [Embarking on a Journey Towards Containerizing Your Workloads](https://www.youtube.com/watch?v=_aFA-p87Eec&index=22&list=PLBgogxgQVM9uSqzLOc66kNIZMUOvnGbU4)

## Troubleshooting

Occasionally the APIs take a few moments to complete. Running `make validate` immediately could potentially appear to fail, but in fact the instances haven't finished initializing. Waiting for a minute or two should resolve the issue.

The setup of this demo **does** take up to **15** minutes. If there is no error the best thing to do is keep waiting. The execution of `make create` should **not** be interrupted.

If you do get an error, it probably makes sense to re-execute the failing script. Occasionally there are network connectivity issues, and retrying will likely work the subsequent time.

**This is not an officially supported Google product**
