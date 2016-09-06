---
layout: post
title:  Sharing Volumes in Docker with NFS
---

After my initial tests with [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/) based on Docker 1.11 I still had some open issues. This docker-swarm was no transparent replacement for a plain Docker host. The main problem was sharing data between containers on different swarm nodes. Some quick tests with the integrated swarm mode for Docker 1.12 showed, that this new mode takes an different approach than the docker-swarm containers before. You need to create special services to utilize it. It doesn't seem to solve the problem with sharing volumes between hosts though.
So I decided to try the most obvious solution next. Docker has no problem with networking. You can create networks to connect multiple containers. So it is possible to use a network file system like NFS to share data between containers.

### Running an NFS server
If you already have an NFS server, you may skip this part. For production it's probably best to use an external NFS service like Amazons EFS. But for some tests, it's easier to start an NFS service as Docker image. For this we will be using the Docker image [cpuguy83/nfs-server ](https://github.com/cpuguy83/docker-nfs-server). First let's create an Docker compose file with the following content:

```
version: '2'
services:
  docker-nfs:
    container_name: nfs-server
    image: cpuguy83/nfs-server
    privileged: true
    volumes:
      - /exports
    command:
      - /exports
```

I assume, that you already have a running docker-swarm. Have a look here [Deploying a multi node Jenkins environment with docker and coreos - Part 1](/Master-Slave-Jenkins-With-Docker-Part1/) for an description of the setup. If you now run docker-compose, you should get the following output:

```
> docker-compose up
Creating network "volumestest_default" with the default driver
Pulling docker-nfs (cpuguy83/nfs-server:latest)...

←[0B-03: Pulling cpuguy83/nfs-server:latest...
←[0B-01: Pulling cpuguy83/nfs-server:latest...
←[3BCreating volumestest_docker-nfs_1latest... : downloaded
Attaching to volumestest_docker-nfs_1
←[36mdocker-nfs_1  |←[0m  * Not starting NFS kernel daemon: no support in current kernel.
←[36mdocker-nfs_1  |←[0m Setting up watches.
←[36mdocker-nfs_1  |←[0m Watches established.
```

So basically it does work, but fails because of missing NFS support in the current kernel. The reason is, that all Docker containers use the Kernel from the Host. So our Docker host needs to have NFS support enabled in the Kernel. You need to load the modules nfs and nfsd for this. To do so, add the following to your cloud-config (file user-data):

```
write_files:
  - path: /etc/modules-load.d/nfs.conf
    content: nfs
  - path: /etc/modules-load.d/nfsd.conf
    content: nfsd
...
coreos:
  units:
  - name: systemd-modules-load.service
    command: restart
```

Now you need to recreate the environment (```vagrant destroy``` && ```vagrant up```). Now run docker-compose again:

```
> docker-compose up
Creating network "volumestest_default" with the default driver
Pulling docker-nfs (cpuguy83/nfs-server:latest)...

←[0B-03: Pulling cpuguy83/nfs-server:latest...
←[0B-02: Pulling cpuguy83/nfs-server:latest...
←[3BCreating nfs-server83/nfs-server:latest... : downloaded
Attaching to nfs-server
←[36mnfs-server    |←[0m  * Exporting directories for NFS kernel daemon...
←[36mnfs-server    |←[0m    ...done.
←[36mnfs-server    |←[0m  * Starting NFS kernel daemon
←[36mnfs-server    |←[0m    ...done.
←[36mnfs-server    |←[0m Setting up watches.
←[36mnfs-server    |←[0m Watches established.
```

Please [read here](https://github.com/cpuguy83/docker-nfs-server/issues/10), if you get the following error message ```rpc.nfsd: unable to resolve ANYADDR:nfs to inet6 address: Servname not supported for ai_socktype```.

### Share your data with NFS
