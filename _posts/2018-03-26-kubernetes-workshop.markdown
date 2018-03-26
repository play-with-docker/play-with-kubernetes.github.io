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

  *  want a cup of tea, obtained by pouring an infusion of tea leaves in a cup.*

  *An infusion is obtained by letting the object steep a few minutes in hot water.*

  *Hot liquid is obtained by pouring it in an appropriate container and setting it on a stove.*

  *Ah, finally, containers! Something we know about. Let's get to work, shall we?*