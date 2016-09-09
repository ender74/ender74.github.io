---
layout: post
title:  Sharing Volumes with Docker Swarm
---

While trying to get [Jenkins](https://jenkins.io/) to run in a docker-swarm installation
I had some problems to share volumes between containers. Usually, you can just define a
data-only container and import volumes from this into other containers. Starting with
docker 1.9 you do not even need to create a data-only container, but can just use named
volumes.

<!-- more -->

I run the following tests with coreos and Docker 1.11. I am using the docker-swarm as a container (not the in Docker 1.12 integrated swarm mode). But theses tests can be run with Docker 1.12 too.

With a plain docker daemon, creating named volumes goes as follows:

```docker volume create --name data```

creates a new named volume. To list all volumes you can use:

```docker volume ls```

and finally to find out more about one volume, the inspect command is your friend:

```docker volume inspect data```

The inspect command shows, that the path for a named volume is much nicer than before.
No need to remember hashcodes anymore.

```
[
    {
        "Name": "data",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/data/_data",
        "Labels": {},
        "Scope": "local"
    }
]
```
To use this named volume, you can just give it's name in the volume argument when
starting your container. For example:

```
$ docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 0
```
The folder is empty right now, but does exist. Let's copy a file into it and check again:

```
$ docker run -i -t -v data:/my-shared-data busybox cp /etc/hostname /my-shared-data
$ docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 8
-rw-r--r--    1 root     root            13 Aug 26 09:23 hostname

$ docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
cb8866fc8def
```
To clean up, you first need to remove all containers, which uses the volume. On my
test system, I will just remove all containers with status "Exited" ```docker rm $(docker ps -aq -f status=exited)```. Afterwards you can remove the volume with ```docker volume rm data```.

Now let's try this with a docker-swarm cluster of three nodes. First we inspect our cluster:

```
$ docker info
Containers: 4
 Running: 4
 Paused: 0
 Stopped: 0
Images: 16
Server Version: swarm/1.2.5
Role: primary
Strategy: spread
Filters: health, port, containerslots, dependency, affinity, constraint
Nodes: 3
 core-01: 172.17.8.101:2375
   ID: LYKZ:YRRL:777L:4J4Z:LCVX:VKNX:OT4N:OHNP:TFAC:TCL2:7C2R:5WL3
   Status: Healthy
   Containers: 2 (2 Running, 0 Paused, 0 Stopped)
   Reserved CPUs: 0 / 1
   Reserved Memory: 0 B / 1.022 GiB
   Labels: kernelversion=4.7.1-coreos, operatingsystem=CoreOS 1151.0.0 (MoreOS), storagedriver=overlay
   UpdatedAt: 2016-08-26T09:33:53Z
   ServerVersion: 1.12.1
 core-02: 172.17.8.102:2375
   ID: QPKC:AY2J:Y5RV:IW3X:U3UK:FGFV:MNYZ:V24O:WZQU:FHQR:VURA:Q6V2
   Status: Healthy
   Containers: 1 (1 Running, 0 Paused, 0 Stopped)
   Reserved CPUs: 0 / 1
   Reserved Memory: 0 B / 1.022 GiB
   Labels: kernelversion=4.7.0-coreos, operatingsystem=CoreOS 1122.0.0 (MoreOS), storagedriver=overlay
   UpdatedAt: 2016-08-26T09:34:22Z
   ServerVersion: 1.11.2
 core-03: 172.17.8.103:2375
   ID: JUO4:TOUW:H7TE:WAI3:I7LH:MTDW:3T23:3TZX:TIAA:WH7E:3Z5C:BDRM
   Status: Healthy
   Containers: 1 (1 Running, 0 Paused, 0 Stopped)
   Reserved CPUs: 0 / 1
   Reserved Memory: 0 B / 1.022 GiB
   Labels: kernelversion=4.7.1-coreos, operatingsystem=CoreOS 1151.0.0 (MoreOS), storagedriver=overlay
   UpdatedAt: 2016-08-26T09:34:00Z
   ServerVersion: 1.12.1
Plugins:
 Volume:
 Network:
Kernel Version: 4.7.1-coreos
Operating System: linux
Architecture: amd64
CPUs: 3
Total Memory: 3.067 GiB
Name: core-01
```

So we have a coreos based cluster with three nodes - core-01 to core-03. If you are interested in this setup, you can read about it here [Deploying a multi node Jenkins environment with docker and coreos - Part 1](/Master-Slave-Jenkins-With-Docker-Part1/).

Let's create a named volume and inspect it:

```
$ docker volume create --name data
data

$ docker volume inspect data
[]
Error: No such volume: data
```

What's going on here? Docker cannot find our just created volume? Let's list all volumes.

```
$ docker volume ls
DRIVER              VOLUME NAME
local               core-01/data
local               core-02/data
local               core-03/data
```

So actually, there were three named volumes created - one on each node. The inspect command doesn't know, which volume we want to inspect. We need to prefix it with the name of the docker host within our swarm:

```
$ docker volume inspect core-01/data
[
    {
        "Name": "data",
        "Driver": "local",
        "Mountpoint": "/var/lib/docker/volumes/data/_data"
    }
]
```
This is not really nice, because it breaks the promise of using the docker-swarm instead of a plain docker host transparently. But let's go on with our tests.

```
$ docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 0
```

Strange enough, this does work. The reason is, that when the busybox container is launched, docker-swarm will choose one of the available nodes to run it on. On this node, there is exactly one named volume data, so everything seems okay.

Let's repeat our test to copy a file into the folder:

```
$ docker run -i -t -v data:/my-shared-data busybox cp /etc/hostname /my-shared-data
$ docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 0

$ docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
cat: can't open '/my-shared-data/hostname': No such file or directory

$ docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
31208bd6c720
```

Not nearly the behaviour I would expect. The fist copy command seems to be fine, but the ls command right afterwards finds no files in the folder. So consequently, the cat cannot find the file either. But executing the same cat again, does find it? According to Albert Einstein I would be mad to expect this behaviour, because I tried the exact same command twice and would expect different results.

The reason lies of cause within the swarm implementation and it's scheduler again. The scheduler selects a node for every container to run it on. In my test, the cp command runs on core-02 and copies the file only there. The ls running on core-01 cannot find it. The first cat runs on core-03 and thus cannot find the file either. But the next cat runs on core-02 again, and now finds the file.

(When you run the test, different nodes might get chosen. But with the default spread scheduler, the effect should be similar and a new node be chosen for subsequent commands.)

To work around this problem, we need to tell the docker-swarm scheduler to run the containers on the same host. To do so, we can use the [affinity filter constraint](https://docs.docker.com/swarm/scheduler/filter/#/use-an-affinity-filter). We give the first container, which copies the file, the name "dataprov". In all subsequent commands we provide an affinity constraint, telling the scheduler to run the container on the same host where the dataprov container was run.

```
$ docker run -i -t -v data:/my-shared-data --name dataprov busybox cp /etc/hostname /my-shared-data
$ docker run -i -t -v data:/my-shared-data -e affinity:container==dataprov busybox ls -l /my-shared-data
total 8
-rw-r--r--    1 root     root            13 Aug 26 11:02 hostname

$ docker run -i -t -v data:/my-shared-data -e affinity:container==dataprov busybox cat /my-shared-data/hostname
088d0ac35fe4

$ docker run -i -t -v data:/my-shared-data -e affinity:container==dataprov busybox cat /my-shared-data/hostname
088d0ac35fe4
```

Now everything runs as expected. But running all containers on one host defeats the purpose of using docker-swarm in the first place.

## Lessons Learned
What did we learn from this? docker-swarm is not a totally transparent replacement for a vanilla docker host. At least when using volumes, you need to be aware that you are running your containers on a cluster.

The described problems occur, because I was using the default volume driver 'local'. As the name suggests, this driver is using more or less only a dumb, local directory. But starting with docker 1.9 there are other plugins available. See the [releasenotes](https://blog.docker.com/2015/11/docker-1-9-production-ready-swarm-multi-host-networking/) for details. By using a more sophisticated volume driver like [flocker](https://docs.clusterhq.com/en/latest/), I could get this probably running.

## Where to go from here?
Right now I see two roads to go further from here. The first is staying with my docker 1.11 swarm and try the flocker plugin. The second possibility would be, to upgrade to docker 1.12 and try the new swarm implementation.
