---
layout: post
title:  Sharing Volumes with docker-swarm 1.12
---

After my initial tests with [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/) based on Docker 1.11 I still had some open issues. This docker-swarm was no transparent replacement for a plain Docker host. The main problem was sharing data between containers on different swarm nodes. Before going deeper and try to fix the issues, I want to try the new Docker 1.12.
With this release comes a new implementation for docker-swarm, which is integrated into the Docker host. I assume, that you already have a development environment setup. Have a look here [Deploying a multi node Jenkins environment with docker and coreos - Part 1](/Master-Slave-Jenkins-With-Docker-Part1/) for an introduction. The main difference is, that we are going to use the current [Alpha Release Channel](https://coreos.com/releases/) of coreos (1153.0.0), which runs Docker 1.12.1.
To start, we grab a fresh copy of the [coreos Vagrant template](https://github.com/coreos/coreos-vagrant), rename the config samples and adjust num_instances to 3.
Before trying to automate the swarm setup with Vagrant, we will try to do the necessary steps manual. For this we will follow the [3 Node Swarm Cluster in 30 seconds](http://www.johnzaccone.io/3-node-cluster-in-30-seconds/) blog entry from John Zaccone.
So start your cluster (```Vagrant up```) and connect to the first box (```Vagrant ssh core-01```). This machine is going to be the swarm master. We need to init the swarm on this machine:

```
core@core-01 ~ $ docker swarm init --advertise-addr 172.17.8.101:2377
Swarm initialized: current node (eay4zvwv68tavuy0anlex35l5) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-45g1yb6yac4e7c9ajftpyr5x62mh2d7nnbi2wfy3lbdj04zqck-b2i3rfsds8391dv4e7dzdtda5 \
    172.17.8.101:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
Easy, wasn't it? Now let's join the two other machines into our swarm. Leave the ssh session with ```exit``` and connect to the second machine (```Vagrant ssh core-02```). Here you copy and paste the command given after swarm init.

```
core@core-02 ~ $     docker swarm join \
<4e7c9ajftpyr5x62mh2d7nnbi2wfy3lbdj04zqck-b2i3rfsds8391dv4e7dzdtda5 \
>     172.17.8.101:2377
This node joined a swarm as a worker.
```
Exit and repeat for core-03. Now it's time to inspect our created swarm and do some basic tests.

```
> set DOCKER_HOST=172.17.8.101:2377

> docker info
Get http://172.17.8.101:2377/v1.24/info: malformed HTTP response "\x15\x03\x01\x00\x02\x02".
* Are you trying to connect to a TLS-enabled daemon without TLS?
```
[Enable the remote API with TLS authentication](https://coreos.com/os/docs/latest/customizing-docker.html#enable-the-remote-api-with-tls-authentication)


Looks like we got a swarm running. Now let's repeat our tests from [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/).

```
> docker volume create --name data

> docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 0

> docker run -i -t -v data:/my-shared-data busybox cp /etc/hostname /my-shared-data

> docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 8
-rw-r--r--    1 root     root            13 Aug 30 12:38 hostname

> docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
d5b19f8a8d4d

> docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
d5b19f8a8d4d
```

Looks great. All worked like expected out of the box. But did the containers run on different nodes of our swarm? To answer this question, we need to inspect the containers. Let's first get their names:

```
> docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS            			PORTS               NAMES
4c8f8b18b099        busybox             "cat /my-shared-data/"   6 minutes ago       Exited (0) 6 minutes ago                 		desperate_jepsen
898ebf0c6e28        busybox             "cat /my-shared-data/"   6 minutes ago       Exited (0) 6 minutes ago                       gigantic_jennings
a4686a701bf3        busybox             "ls -l /my-shared-dat"   6 minutes ago       Exited (0) 6 minutes ago                       pedantic_saha
d5b19f8a8d4d        busybox             "cp /etc/hostname /my"   6 minutes ago       Exited (0) 6 minutes ago                       hungry_chandrasekhar
36e9d1bebcff        busybox             "ls -l /my-shared-dat"   7 minutes ago       Exited (0) 7 minutes ago                       hopeful_babbage
d3eaf4d9f487        hello-world         "/hello"                 9 minutes ago       Exited (0) 9 minutes ago                       evil_shockley
```

We can now use the inspect command, to query the hostname for each container:
```
> docker inspect desperate_jepsen
...
"Config": {
    "Hostname": "4c8f8b18b099",
    "Domainname": "",
    ...
},
...
> docker inspect gigantic_jennings
...
"Config": {
    "Hostname": "898ebf0c6e28",
    "Domainname": "",
    ...
},
...
> docker inspect pedantic_saha
...
"Config": {
    "Hostname": "a4686a701bf3",
    "Domainname": "",
    ...
},
...
> docker inspect hungry_chandrasekhar
...
"Config": {
    "Hostname": "4c8f8b18b099",
    "Domainname": "",
    ...
},
...
```
