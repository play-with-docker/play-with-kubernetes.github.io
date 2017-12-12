---
layout: post
title:  "Kubernetes for Beginners"
date:   2017-12-07
author: "@jpetazzo"
tags: [beginner, linux, operations, kubernetes, developer]
categories: beginner
terms: 1
---

In this lab, you will get a hands-on overview of Kubernetes concepts and functionality. In the right pane, you will see two command line terminals, representing two different nodes. 

> **Difficulty**: Beginner (assumes basic familiarity with Docker concepts)

> **Tasks**:
>

> * [Task 0: Prerequisites](#Task_0)
> * [Task 1: Run some simple Docker containers](#Task_1)
> * [Task 2: Package and run a custom app using Docker](#Task_2)
> * [Task 3: Modify a Running Website](#Task_3)

## <a name="task0"></a>Task 0: Prerequisites

- Basic familiarity with Docker. `docker run`, `docker ps`, `docker build` etc. ideally, you know how to write a Dockerfile and build it. If you aren't familiar with Docker at all, check out the [beginner tutorial](training.play-with-docker.com/beginner-linux/) on the Play with Docker site.

- Basic familiarity with the Linux commandline, navigating directories, editing files, basic bash knowledge.

### Notes
- We will run almost all the commands on the first node, node1
- If a command is highlighted in gray, you can click on it and it will autopopulate in the appropriate terminal. For instance:
```.term1
kubectl --help
```

## Brand new versions!

- Kubernetes 1.8
- Docker Engine 17.11
- Docker Compose 1.17


.exercise[

- Check all installed versions:
  ```bash
  kubectl version
  docker version
  docker-compose -v
  ```

]

# Sample application

- Visit the GitHub repository with all the materials of this workshop:
  <br/>https://github.com/jpetazzo/container.training

- The application is in the [dockercoins](
  https://github.com/jpetazzo/container.training/tree/master/dockercoins)
  subdirectory

- Let's look at the general layout of the source code:

  there is a Compose file [docker-compose.yml](
  https://github.com/jpetazzo/container.training/blob/master/dockercoins/docker-compose.yml) ...

  ... and 4 other services, each in its own directory:

  - `rng` = web service generating random bytes
  - `hasher` = web service computing hash of POSTed data
  - `worker` = background process using `rng` and `hasher`
  - `webui` = web interface to watch progress


## Compose file format version

*Particularly relevant if you have used Compose before...*

- Compose 1.6 introduced support for a new Compose file format (aka "v2")

- Services are no longer at the top level, but under a `services` section

- There has to be a `version` key at the top level, with value `"2"` (as a string, not an integer)

- Containers are placed on a dedicated network, making links unnecessary

- There are other minor differences, but upgrade is easy and straightforward

## Links, naming, and service discovery

- Containers can have network aliases (resolvable through DNS)

- Compose file version 2+ makes each container reachable through its service name

- Compose file version 1 did require "links" sections

- Our code can connect to services using their short name

  (instead of e.g. IP address or FQDN)

- Network aliases are automatically namespaced

  (i.e. you can have multiple apps declaring and using a service named `database`)
  
  ## Example in `worker/worker.py`

```python
redis = Redis("`redis`")


def get_random_bytes():
    r = requests.get("http://`rng`/32")
    return r.content


def hash_bytes(data):
    r = requests.post("http://`hasher`/",
                      data=data,
                      headers={"Content-Type": "application/octet-stream"})
```

(Full source code available [here](
https://github.com/jpetazzo/container.training/blob/8279a3bce9398f7c1a53bdd95187c53eda4e6435/dockercoins/worker/worker.py#L17
))

## What's this application?




- It is a DockerCoin miner! .emoji[💰🐳📦🚢]




- No, you can't buy coffee with DockerCoins




- How DockerCoins works:

  - `worker` asks to `rng` to generate a few random bytes

  - `worker` feeds these bytes into `hasher`

  - and repeat forever!

  - every second, `worker` updates `redis` to indicate how many loops were done

  - `webui` queries `redis`, and computes and exposes "hashing speed" in your browser

## Getting the application source code

- We will clone the GitHub repository

- The repository also contains scripts and tools that we will use through the workshop

.exercise[

<!--
```bash
if [ -d container.training ]; then
  mv container.training container.training.$$
fi
```
-->

- Clone the repository on `node1`:
  ```.term1
  git clone git://github.com/jpetazzo/container.training
  ```

]

(You can also fork the repository on GitHub and clone your fork if you prefer that.)

# Running the application

Without further ado, let's start our application.

.exercise[

- Go to the `dockercoins` directory, in the cloned repo:
  ```bash
  cd ~/container.training/dockercoins
  ```

- Use Compose to build and run all containers:
  ```bash
  docker-compose up
  ```

<!--
```longwait units of work done```
```keys ^C```
-->

]

Compose tells Docker to build all container images (pulling
the corresponding base images), then starts all containers,
and displays aggregated logs.

## Lots of logs

- The application continuously generates logs

- We can see the `worker` service making requests to `rng` and `hasher`

- Let's put that in the background

.exercise[

- Stop the application by hitting `^C`

]

- `^C` stops all containers by sending them the `TERM` signal

- Some containers exit immediately, others take longer
  <br/>(because they don't handle `SIGTERM` and end up being killed after a 10s timeout)

## Restarting in the background

- Many flags and commands of Compose are modeled after those of `docker`

.exercise[

- Start the app in the background with the `-d` option:
  ```bash
  docker-compose up -d
  ```

- Check that our app is running with the `ps` command:
  ```bash
  docker-compose ps
  ```

]

`docker-compose ps` also shows the ports exposed by the application.



## Viewing logs

- The `docker-compose logs` command works like `docker logs`

.exercise[

- View all logs since container creation and exit when done:
  ```bash
  docker-compose logs
  ```

- Stream container logs, starting at the last 10 lines for each container:
  ```bash
  docker-compose logs --tail 10 --follow
  ```

<!--
```wait units of work done```
```keys ^C```
-->

]

Tip: use `^S` and `^Q` to pause/resume log output.





## Upgrading from Compose 1.6

.warning[The `logs` command has changed between Compose 1.6 and 1.7!]

- Up to 1.6

  - `docker-compose logs` is the equivalent of `logs --follow`

  - `docker-compose logs` must be restarted if containers are added

- Since 1.7

  - `--follow` must be specified explicitly

  - new containers are automatically picked up by `docker-compose logs`



## Connecting to the web UI

- The `webui` container exposes a web dashboard; let's view it

.exercise[

- With a web browser, connect to `node1` on port 8000

- Remember: the `nodeX` aliases are valid only on the nodes themselves

- In your browser, you need to enter the IP address of your node

<!-- ```open http://node1:8000``` -->

]

A drawing area should show up, and after a few seconds, a blue
graph will appear.


## If the graph doesn't load

If you just see a `Page not found` error, it might be because your
Docker Engine is running on a different machine. This can be the case if:

- you are using the Docker Toolbox

- you are using a VM (local or remote) created with Docker Machine

- you are controlling a remote Docker Engine

When you run DockerCoins in development mode, the web UI static files
are mapped to the container using a volume. Alas, volumes can only
work on a local environment, or when using Docker4Mac or Docker4Windows.

How to fix this?

Edit `dockercoins.yml` and comment out the `volumes` section, and try again.





## Why does the speed seem irregular?

- It *looks like* the speed is approximately 4 hashes/second

- Or more precisely: 4 hashes/second, with regular dips down to zero

- Why?






- The app actually has a constant, steady speed: 3.33 hashes/second
  <br/>
  (which corresponds to 1 hash every 0.3 seconds, for *reasons*)

- Yes, and?





## The reason why this graph is *not awesome*

- The worker doesn't update the counter after every loop, but up to once per second

- The speed is computed by the browser, checking the counter about once per second

- Between two consecutive updates, the counter will increase either by 4, or by 0

- The perceived speed will therefore be 4 - 4 - 4 - 0 - 4 - 4 - 0 etc.

- What can we conclude from this?






- Jérôme is clearly incapable of writing good frontend code



## Scaling up the application

- Our goal is to make that performance graph go up (without changing a line of code!)




- Before trying to scale the application, we'll figure out if we need more resources

  (CPU, RAM...)

- For that, we will use good old UNIX tools on our Docker node



## Looking at resource usage

- Let's look at CPU, memory, and I/O usage

.exercise[

- run `top` to see CPU and memory usage (you should see idle cycles)

<!--
```bash top```

```wait Tasks```
```keys ^C```
-->

- run `vmstat 1` to see I/O usage (si/so/bi/bo)
  <br/>(the 4 numbers should be almost zero, except `bo` for logging)

<!--
```bash vmstat 1```

```wait memory```
```keys ^C```
-->

]

We have available resources.

- Why?
- How can we use them?



## Scaling workers on a single node

- Docker Compose supports scaling
- Let's scale `worker` and see what happens!

.exercise[

- Start one more `worker` container:
  ```bash
  docker-compose scale worker=2
  ```

- Look at the performance graph (it should show a x2 improvement)

- Look at the aggregated logs of our containers (`worker_2` should show up)

- Look at the impact on CPU load with e.g. top (it should be negligible)

]



## Adding more workers

- Great, let's add more workers and call it a day, then!

.exercise[

- Start eight more `worker` containers:
  ```bash
  docker-compose scale worker=10
  ```

- Look at the performance graph: does it show a x10 improvement?

- Look at the aggregated logs of our containers

- Look at the impact on CPU load and memory usage

]



# Identifying bottlenecks

- You should have seen a 3x speed bump (not 10x)

- Adding workers didn't result in linear improvement

- *Something else* is slowing us down



- ... But what?




- The code doesn't have instrumentation

- Let's use state-of-the-art HTTP performance analysis!
  <br/>(i.e. good old tools like `ab`, `httping`...)



## Accessing internal services

- `rng` and `hasher` are exposed on ports 8001 and 8002

- This is declared in the Compose file:

  ```yaml
    ...
    rng:
      build: rng
      ports:
      - "8001:80"

    hasher:
      build: hasher
      ports:
      - "8002:80"
    ...
  ```



## Measuring latency under load

We will use `httping`.

.exercise[

- Check the latency of `rng`:
  ```bash
  httping -c 10 localhost:8001
  ```

- Check the latency of `hasher`:
  ```bash
  httping -c 10 localhost:8002
  ```

]

`rng` has a much higher latency than `hasher`.



## Let's draw hasty conclusions

- The bottleneck seems to be `rng`

- *What if* we don't have enough entropy and can't generate enough random numbers?

- We need to scale out the `rng` service on multiple machines!

Note: this is a fiction! We have enough entropy. But we need a pretext to scale out.

(In fact, the code of `rng` uses `/dev/urandom`, which never runs out of entropy...
<br/>
...and is [just as good as `/dev/random`](http://www.slideshare.net/PacSecJP/filippo-plain-simple-reality-of-entropy).)



## Clean up

- Before moving on, let's remove those containers

.exercise[

- Tell Compose to remove everything:
  ```bash
  docker-compose down
  ```

]
# Kubernetes concepts

- Kubernetes is a container management system

- It runs and manages containerized applications on a cluster



- What does that really mean?



## Basic things we can ask Kubernetes to do



- Start 5 containers using image `atseashop/api:v1.3`



- Place an internal load balancer in front of these containers



- Start 10 containers using image `atseashop/webfront:v1.3`



- Place a public load balancer in front of these containers



- It's Black Friday (or Christmas), traffic spikes, grow our cluster and add containers



- New release! Replace my containers with the new image `atseashop/webfront:v1.4`



- Keep processing requests during the upgrade; update my containers one at a time



## Other things that Kubernetes can do for us

- Basic autoscaling

- Blue/green deployment, canary deployment

- Long running services, but also batch (one-off) jobs

- Overcommit our cluster and *evict* low-priority jobs

- Run services with *stateful* data (databases etc.)

- Fine-grained access control defining *what* can be done by *whom* on *which* resources

- Integrating third party services (*service catalog*)

- Automating complex tasks (*operators*)



## Kubernetes architecture





![haha only kidding](images/k8s-arch1.png)



## Kubernetes architecture

- Ha ha ha ha

- OK, I was trying to scare you, it's much simpler than that ❤️





![that one is more like the real thing](images/k8s-arch2.png)



## Credits

- The first schema is a Kubernetes cluster with storage backed by multi-path iSCSI

  (Courtesy of [Yongbok Kim](https://www.yongbok.net/blog/))

- The second one is a simplified representation of a Kubernetes cluster

  (Courtesy of [Imesh Gunaratne](https://medium.com/containermind/a-reference-architecture-for-deploying-wso2-middleware-on-kubernetes-d4dee7601e8e))



## Kubernetes architecture: the master

- The Kubernetes logic (its "brains") is a collection of services:

  - the API server (our point of entry to everything!)
  - core services like the scheduler and controller manager
  - `etcd` (a highly available key/value store; the "database" of Kubernetes)

- Together, these services form what is called the "master"

- These services can run straight on a host, or in containers
  <br/>
  (that's an implementation detail)

- `etcd` can be run on separate machines (first schema) or co-located (second schema)

- We need at least one master, but we can have more (for high availability)



## Kubernetes architecture: the nodes

- The nodes executing our containers run another collection of services:

  - a container Engine (typically Docker)
  - kubelet (the "node agent")
  - kube-proxy (a necessary but not sufficient network component)

- Nodes were formerly called "minions"

- It is customary to *not* run apps on the node(s) running master components

  (Except when using small development clusters) 



## Do we need to run Docker at all?

No!



- By default, Kubernetes uses the Docker Engine to run containers

- We could also use `rkt` ("Rocket") from CoreOS

- Or leverage other pluggable runtimes through the *Container Runtime Interface*

  (like CRI-O, or containerd)



## Do we need to run Docker at all?

Yes!



- In this workshop, we run our app on a single node first

- We will need to build images and ship them around

- We can do these things without Docker
  <br/>
  (and get diagnosed with NIH¹ syndrome)

- Docker is still the most stable container engine today
  <br/>
  (but other options are maturing very quickly)

.footnote[¹[Not Invented Here](https://en.wikipedia.org/wiki/Not_invented_here)]



## Do we need to run Docker at all?

- On our development environments, CI pipelines ... :

  *Yes, almost certainly*

- On our production servers:

  *Yes (today)*

  *Probably not (in the future)*

.footnote[More information about CRI [on the Kubernetes blog](http://blog.kubernetes.io/2016/12/container-runtime-interface-cri-in-kubernetes.html)]



## Kubernetes resources

- The Kubernetes API defines a lot of objects called *resources*

- These resources are organized by type, or `Kind` (in the API)

- A few common resource types are:

  - node (a machine — physical or virtual — in our cluster)
  - pod (group of containers running together on a node)
  - service (stable network endpoint to connect to one or multiple containers)
  - namespace (more-or-less isolated group of things)
  - secret (bundle of sensitive data to be passed to a container)
 
  And much more! (We can see the full list by running `kubectl get`)





![Node, pod, container](images/k8s-arch3-thanks-weave.png)

(Diagram courtesy of Weave Works, used with permission.)

# Declarative vs imperative

- Our container orchestrator puts a very strong emphasis on being *declarative*

- Declarative:

  *I would like a cup of tea.*

- Imperative:

  *Boil some water. Pour it in a teapot. Add tea leaves. Steep for a while. Serve in cup.*



- Declarative seems simpler at first ... 



- ... As long as you know how to brew tea



## Declarative vs imperative

- What declarative would really be:

  *I want a cup of tea, obtained by pouring an infusion¹ of tea leaves in a cup.*



  *¹An infusion is obtained by letting the object steep a few minutes in hot² water.*



  *²Hot liquid is obtained by pouring it in an appropriate container³ and setting it on a stove.*



  *³Ah, finally, containers! Something we know about. Let's get to work, shall we?*



.footnote[Did you know there was an [ISO standard](https://en.wikipedia.org/wiki/ISO_3103)
specifying how to brew tea?]



## Declarative vs imperative

- Imperative systems:

  - simpler

  - if a task is interrupted, we have to restart from scratch

- Declarative systems:

  - if a task is interrupted (or if we show up to the party half-way through),
    we can figure out what's missing and do only what's necessary

  - we need to be able to *observe* the system

  - ... and compute a "diff" between *what we have* and *what we want*
