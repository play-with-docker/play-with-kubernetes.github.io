---
layout: post
title:  "Kubernetes for Beginners"
date:   2017-12-07
author: "@jpetazzo"
tags: [beginner, linux, operations, kubernetes, developer]
categories: beginner
terms: 3
---

### Our sample application
Visit the GitHub repository with all the materials of this workshop: 
(https://github.com/jpetazzo/container.training)[https://github.com/jpetazzo/container.training]

The application is in the (dockercoins)[https://github.com/jpetazzo/container.training/tree/master/dockercoins] subdirectory.

Let's look at the general layout of the source code:

there is a Compose file docker-compose.yml ...

... and 4 other services, each in its own directory:

`rng` = web service generating random bytes
`hasher` = web service computing hash of POSTed data
`worker` = background process using `rng` and `hasher`
`webui` = web interface to watch progress

### What's this application?
* It is a DockerCoin miner! 💰🐳📦🚢

* No, you can't buy coffee with DockerCoins

* How DockerCoins works:

  * `worker` asks to `rng` to generate a few random bytes

  * `worker` feeds these bytes into `hasher`

  * and repeat forever!

  * every second, `worker` updates `redis` to indicate how many loops were done

  * `webui` queries `redis`, and computes and exposes "hashing speed" in your browser

### Getting the application source code
* We will clone the GitHub repository

* The repository also contains scripts and tools that we will use through the workshop

```.term1
git clone https://github.com/jpetazzo/container.training/
```

(You can also fork the repository on GitHub and clone your fork if you prefer that.)

## Running the application

### Running the application

* Go to the dockercoins directory, in the cloned repo:

```.term1
cd ~/container.training/dockercoins
```

* Use Compose to build and run all containers:

```.term1
docker-compose up
```

Compose tells Docker to build all container images (pulling the corresponding base images), then starts all containers, and displays aggregated logs.

### Lots of logs

* The application continuously generates logs

* We can see the worker service making requests to rng and hasher

*  Let's put that in the background

* Stop the application by hitting `^C`
`^C` stops all containers by sending them the `TERM` signal

* Some containers exit immediately, others take longer (because they don't handle `SIGTERM` and end up being killed after a 10s timeout)

### Connecting to the web UI

* The `webui` container exposes a web dashboard; let's view it

* With a web browser, connect to `node1` on this link: port 8000 <!-- TODO add link --> (created when you ran the application)

### Clean up

* Before moving on, let's remove those containers. Tell Compose to remove everything:

```.term1
docker-compose down
```

## Kubernetes concepts

### Kubernetes concepts

* Kubernetes is a container management system

* It runs and manages containerized applications on a cluster

* What does that really mean?

### Basic things we can ask Kubernetes to do

* Start 5 containers using image `atseashop/api:v1.3`

* Place an internal load balancer in front of these containers

* Start 10 containers using image `atseashop/webfront:v1.3`

* Place a public load balancer in front of these containers

* It's Black Friday (or Christmas), traffic spikes, grow our cluster and add containers

* New release! Replace my containers with the new image `atseashop/webfront:v1.4`

* Keep processing requests during the upgrade; update my containers one at a time

### Other things that Kubernetes can do for us

* Basic autoscaling

* Blue/green deployment, canary deployment

* Long running services, but also batch (one-off) jobs

* Overcommit our cluster and evict low-priority jobs

* Run services with stateful data (databases etc.)

* Fine-grained access control defining what can be done by whom on which resources

* Integrating third party services (service catalog)

* Automating complex tasks (operators)

### Kubernetes architecture

<!-- TODO: Come up with diagrams -->

### Kubernetes architecture: the master
* The Kubernetes logic (its "brains") is a collection of services:

  * the API server (our point of entry to everything!)
  * core services like the scheduler and controller manager
  * etcd (a highly available key/value store; the "database" of Kubernetes)

*Together, these services form what is called the "master"

* These services can run straight on a host, or in containers (that's an implementation detail)

* etcd can be run on separate machines (first schema) or co-located (second schema)

* We need at least one master, but we can have more (for high availability)

### Kubernetes architecture: the nodes

* The nodes executing our containers run another collection of services:

  * a container Engine (typically Docker)
  * kubelet (the "node agent")
  * kube-proxy (a necessary but not sufficient network component)

* Nodes were formerly called "minions"

* It is customary to not run apps on the node(s) running master components (Except when using small development clusters)

## Kubernetes resources

* The Kubernetes API defines a lot of objects called resources

* These resources are organized by type, or Kind (in the API)

* A few common resource types are:

  * node (a machine — physical or virtual — in our cluster)
  * pod (group of containers running together on a node)
  * service (stable network endpoint to connect to one or multiple containers)
  * namespace (more-or-less isolated group of things)
  * secret (bundle of sensitive data to be passed to a container)

* And much more! (We can see the full list by running kubectl get)

<!-- TODO: Add diagrams illustrating these concepts -->

## Declarative vs imperative

### Declarative vs imperative
* Our container orchestrator puts a very strong emphasis on being declarative

* Declarative:

  *I would like a cup of tea.*

* Imperative:

  *Boil some water. Pour it in a teapot. Add tea leaves. Steep for a while. Serve in cup.*

* Declarative seems simpler at first ...

* ... As long as you know how to brew tea

#### What declarative would really be:

  *I want a cup of tea, obtained by pouring an infusion of tea leaves in a cup.*

  *An infusion is obtained by letting the object steep a few minutes in hot water.*

  *Hot liquid is obtained by pouring it in an appropriate container and setting it on a stove.*

  *Ah, finally, containers! Something we know about. Let's get to work, shall we?*

### Summary of declarative vs imperative

* Imperative systems:

  * simpler

  * if a task is interrupted, we have to restart from scratch

* Declarative systems:

  * if a task is interrupted (or if we show up to the party half-way through), we can figure out what's missing and do only what's necessary

  * we need to be able to *observe* the system

  * ... and compute a "diff" between *what we have and what we want*

### Declarative vs imperative in Kubernetes

* Virtually everything we create in Kubernetes is created from a `spec`

* Watch for the `spec` fields in the YAML files later!

* The `spec` describes *how we want the thing to be*

* Kubernetes will *reconcile* the current state with the spec (technically, this is done by a number of *controllers*)

* When we want to change some resource, we update the `spec`

* Kubernetes will then *converge* that resource

## Kubernetes network model

### Kubernetes network model
* TL,DR:

  *Our cluster (nodes and pods) is one big flat IP network.*

* In detail:

  * all nodes must be able to reach each other, without NAT

  * all pods must be able to reach each other, without NAT

  * pods and nodes must be able to reach each other, without NAT

  * each pod is aware of its IP address (no NAT)

* Kubernetes doesn't mandate any particular implementation

## Kubernetes network model: the good

* Everything can reach everything

* No address translation

* No port translation

* No new protocol

* Pods cannot move from a node to another and keep their IP address

* IP addresses don't have to be "portable" from a node to another (We can use e.g. a subnet per node and use a simple routed topology)

* The specification is simple enough to allow many various implementations

## Kubernetes network model: the less good

* Everything can reach everything

  * if you want security, you need to add network policies

  * the network implementation that you use needs to support them

* There are literally dozens of implementations out there (15 are listed in the Kubernetes documentation)

* It looks like you have a level 3 network, but it's only level 4 (The spec requires UDP and TCP, but not port ranges or arbitrary IP packets)

* `kube-proxy` is on the data path when connecting to a pod or container, and it's not particularly fast (relies on userland proxying or iptables)

### Kubernetes network model: in practice

* The nodes that we are using have been set up to use Weave

* We don't endorse Weave in a particular way, it just Works For Us

* Don't worry about the warning about kube-proxy performance

* Unless you:

  * routinely saturate 10G network interfaces

  * count packet rates in millions per second

  * run high-traffic VOIP or gaming platforms

  * do weird things that involve millions of simultaneous connections (in which case you're already familiar with kernel tuning)

## First contact with `kubectl`

* `kubectl` is (almost) the only tool we'll need to talk to Kubernetes

* It is a rich CLI tool around the Kubernetes API (Everything you can do with `kubectl`, you can do directly with the API)

<!-- TODO: Is this necessary and true? -->
* On our machines, there is a `~/.kube/config` file with:

  * the Kubernetes API address

  * the path to our TLS certificates used to authenticate

* You can also use the `--kubeconfig` flag to pass a config file

* Or directly `--server`, `--user`, etc.

* `kubectl` can be pronounced "Cube C T L", "Cube cuttle", "Cube cuddle"...

### `kubectl get`

* Let's look at our Node resources with kubectl get!

* Look at the composition of our cluster:

  ```.term
  kubectl get node
  ```
* These commands are equivalent

  ```
  kubectl get no
  kubectl get node
  kubectl get nodes
  ```

### Obtaining machine-readable output

* `kubectl get` can output JSON, YAML, or be directly formatted

* Give us more info about the nodes:

  ```.term1
  kubectl get nodes -o wide
  ```

* Let's have some YAML:
  ```.term1
  kubectl get no -o yaml
  ```
  See that kind: List at the end? It's the type of our result!

### (Ab)using `kubectl` and `jq`

* It's super easy to build custom reports

* Show the capacity of all our nodes as a stream of JSON objects:
  ```.term1
  kubectl get nodes -o json | 
        jq ".items[] | {name:.metadata.name} + .status.capacity"
  ```

### What's available?

* `kubectl` has pretty good introspection facilities

* We can list all available resource types by running `kubectl get`

* We can view details about a resource with:
  ```
  kubectl describe type/name
  kubectl describe type name
  ```

* We can view the definition for a resource type with:
  ```
  kubectl explain type
  ```

Each time, `type` can be singular, plural, or abbreviated type name.

### Services

* A service is a stable endpoint to connect to "something" (In the initial proposal, they were called "portals")

* List the services on our cluster with one of these commands:
  ```
  kubectl get services
  kubectl get svc
  ```

There is already one service on our cluster: the Kubernetes API itself.

### ClusterIP services

* A `ClusterIP` service is internal, available from the cluster only

* This is useful for introspection from within containers

* Try to connect to the API: <!-- TODO: Check this is accurate -->

  ```.term1
  curl -k https://10.96.0.1
  ```

  * `-k` is used to skip certificate verification
  * Make sure to replace 10.96.0.1 with the CLUSTER-IP shown by `$ kubectl get svc`

The error that we see is expected: the Kubernetes API requires authentication.

### Listing running containers

* Containers are manipulated through pods

* A pod is a group of containers:

  * running together (on the same node)

  * sharing resources (RAM, CPU; but also network, volumes)

* List pods on our cluster:

  ```.term1
  kubectl get pods
  ```
*These are not the pods you're looking for*. But where are they?!?

### Namespaces

* Namespaces allow us to segregate resources

* List the namespaces on our cluster with one of these commands:

```
kubectl get namespaces
kubectl get namespace
kubectl get ns
```
*You know what ... This `kube-system` thing looks suspicious.*

### Accessing namespaces
* By default, `kubectl` uses the `default` namespace

* We can switch to a different namespace with the `-n` option

* List the pods in the `kube-system` namespace:
  ```.term1
  kubectl -n kube-system get pods
  ```
*Ding ding ding ding ding!*

### What are all these pods?

* `etcd` is our etcd server

* `kube-apiserver` is the API server

* `kube-controller-manager` and `kube-scheduler` are other master components

* `kube-dns` is an additional component (not mandatory but super useful, so it's there)

* `kube-proxy` is the (per-node) component managing port mappings and such

* `weave` is the (per-node) component managing the network overlay <!-- Note: Mention of Weave -->

* the `READY` column indicates the number of containers in each pod

* the pods with a name ending with `-node1` are the master components (they have been specifically "pinned" to the master node)

## Running our first containers on Kubernetes

### Running our first containers on Kubernetes
* First things first: we cannot run a container

* We are going to run a pod, and in that pod there will be a single container

* In that container in the pod, we are going to run a simple ping command

* Then we are going to start additional copies of the pod

### Starting a simple pod with `kubectl run`

* We need to specify at least a name and the image we want to use

* Let's ping `goo.gl`

```.term1
kubectl run pingpong --image alpine ping goo.gl
```

* OK, what just happened?

### Behind the scenes of `kubectl run`

* Let's look at the resources that were created by `kubectl run`

* List most resource types:

```.term1
kubectl get all
```

We should see the following things:

* `deploy/pingpong` (the *deployment* that we just created)
* `rs/pingpong-xxxx` (a *replica set* created by the deployment)
* `po/pingpong-yyyy` (a *pod* created by the replica set)

### What are these different things?

* A *deployment* is a high-level construct

  * allows scaling, rolling updates, rollbacks

  * multiple deployments can be used together to implement a [canary deployment](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)

  * delegates pods management to *replica sets*

* A *replica set* is a low-level construct

  * makes sure that a given number of identical pods are running

  * allows scaling

  * rarely used directly

* A *replication controller* is the (deprecated) predecessor of a replica set

### Our `pingpong` deployment

* `kubectl run` created a *deployment*, `deploy/pingpong`

* That deployment created a *replica set*, `rs/pingpong-xxxx`

* That *replica set* created a *pod*, `po/pingpong-yyyy`

* We'll see later how these folks play together for:

  * scaling

  * high availability

  * rolling updates

### Viewing container output

* Let's use the `kubectl logs` command

* We will pass either a *pod name*, or a *type/name*
(E.g. if we specify a deployment or replica set, it will get the first pod in it)

* Unless specified otherwise, it will only show logs of the first container in the pod
(Good thing there's only one in ours!)

* View the result of our ping command:

```.term1
kubectl logs deploy/pingpong
```

### Streaming logs in real time
* Just like `docker logs`, `kubectl logs` supports convenient options:

  * `-f/--follow` to stream logs in real time (à la tail `-f`)

  * `--tail` to indicate how many lines you want to see (from the end)

  * `--since` to get logs only after a given timestamp

* View the latest logs of our ping command:

```.term1
kubectl logs deploy/pingpong --tail 1 --follow
```

### Scaling our application

* We can create additional copies of our container (or rather our pod) with `kubectl scale`

* Scale our pingpong deployment:
```.term1
kubectl scale deploy/pingpong --replicas 8
```
> Note: what if we tried to scale `rs/pingpong-xxxx`? We could! But the *deployment* would notice it right away, and scale back to the initial level.

### Resilience

* The deployment pingpong watches its replica set

* The replica set ensures that the right number of pods are running

* What happens if pods disappear?

* In a separate window, list pods, and keep watching them:
```.term1
kubectl get pods -w
```

* Destroy a pod:
```.term1
kubectl delete pod pingpong-yyyy
```

### What if we wanted something different?
* What if we wanted to start a "one-shot" container that *doesn't* get restarted?

* We could use `kubectl run --restart=OnFailure` or `kubectl run --restart=Never`

* These commands would create *jobs* or *pods* instead of *deployments*

* Under the hood, `kubectl run` invokes "generators" to create resource descriptions

* We could also write these resource descriptions ourselves (typically in YAML), 
and create them on the cluster with `kubectl apply -f` (discussed later)

* With `kubectl run --schedule=`..., we can also create *cronjobs*

### Viewing logs of multiple pods

* When we specify a deployment name, only one single pod's logs are shown

* We can view the logs of multiple pods by specifying a *selector*

* A selector is a logic expression using *labels*

* Conveniently, when `you kubectl run somename`, the associated objects have a `run=somename` label

* View the last line of log from all pods with the `run=pingpong` label:
```.term1
kubectl logs -l run=pingpong --tail 1
```

* Unfortunately, `--follow` cannot (yet) be used to stream the logs from multiple containers.

## Exposing containers
### Exposing containers

* `kubectl expose` creates a *service* for existing pods

* A *service* is a stable address for a pod (or a bunch of pods)

* If we want to connect to our pod(s), we need to create a *service*

* Once a service is created, `kube-dns` will allow us to resolve it by name (i.e. after creating service `hello`, the name `hello` will resolve to something)

* There are different types of services, detailed on the following slides:

`ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`

## Basic service types

* `ClusterIP` (default type)

  a virtual IP address is allocated for the service (in an internal, private range)
  this IP address is reachable only from within the cluster (nodes and pods)
  our code can connect to the service using the original port number

* `NodePort`

  a port is allocated for the service (by default, in the 30000-32768 range)
  that port is made available on all our nodes and anybody can connect to it
  our code must be changed to connect to that new port number

These service types are always available.

Under the hood: `kube-proxy` is using a userland proxy and a bunch of `iptables` rules.

### More service types

* `LoadBalancer`
<!--TODO: Check if LoadBalancer is available in Docker for Desktop. Currently available in EE -->
  * an external load balancer is allocated for the service
  * the load balancer is configured accordingly (e.g.: a `NodePort` service is created, and the load balancer sends traffic to that port)

* `ExternalName`

  * the DNS entry managed by `kube-dns` will just be a `CNAME` to a provided record
  * no port, no IP address, no nothing else is allocated

### Running containers with open ports

* Since ping doesn't have anything to connect to, we'll have to run something else

* Start a bunch of ElasticSearch containers:
```.term1
kubectl run elastic --image=elasticsearch:2 --replicas=7
```

* Watch them being started:

```.term1
kubectl get pods -w
```

The `-w` option "watches" events happening on the specified resources.

Note: please DO NOT call the service `search`. It would collide with the TLD.

### Exposing our deployment

* We'll create a default `ClusterIP` service

* Expose the ElasticSearch HTTP API port:

```.term
kubectl expose deploy/elastic --port 9200
```

* Look up which IP address was allocated:

```.term1
kubectl get svc
```
### Services are layer 4 constructs

* You can assign IP addresses to services, but they are still *layer 4* (i.e. a service is not an IP address; it's an IP address + protocol + port)

* This is caused by the current implementation of `kube-proxy` (it relies on mechanisms that don't support layer 3)

* As a result: *you have to* indicate the port number for your service

* Running services with arbitrary port (or port ranges) requires hacks (e.g. host networking mode)

### Testing our service

* We will now send a few HTTP requests to our ElasticSearch pods

* Let's obtain the IP address that was allocated for our service, *programatically*:
```.term1
IP=$(kubectl get svc elastic -o go-template --template '{{ .spec.clusterIP }}')
```

* Send a few requests:

```.term1
curl http://$IP:9200/
```

Our requests are load balanced across multiple pods.

## Our app on Kube

### What's on the menu?
In this part, we will:

  * **build** images for our app,

  * **ship** these images with a registry,

  * **run** deployments using these images,

  * expose these deployments so they can communicate with each other,

  * expose the web UI so we can access it from outside.



