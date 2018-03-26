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

