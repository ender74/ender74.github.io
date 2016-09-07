---
layout: post
title:  Sharing Volumes in Docker Swarm with NFS
---

After my initial tests with [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/) based on Docker 1.11 I still had some open issues. This docker-swarm was no transparent replacement for a plain Docker host. The main problem was sharing data between containers on different swarm nodes. Some quick tests with the integrated swarm mode for Docker 1.12 showed, that this new mode takes an different approach than the docker-swarm containers before. You need to create special services to utilize it. It doesn't seem to solve the problem with sharing volumes between hosts though and does not work with docker-compose yet.

So I decided to try a more straightforward solution next. Docker has no problem with networking. You can create networks to connect multiple containers. So it is surely possible to use a network file system like NFS to share data between containers.

TLDR; You can get an running Vagrant Template in my [Github Repository](https://github.com/ender74/coreos-vagrant.git). If you start with this template, you should be able to just [run the tests](#running-the-tests). Read on to find out more about this configuration.

### Running an NFS server
If you already have an NFS server, you may skip this part. For production it's probably best to use an external file service like NFS or Amazons EFS. But for some tests, it's easier to start the NFS service with our virtual Vagrant Environment. For this we will be editing the cloud-config (user-data file) as follows. First create some files:

```
write-files:
  - path: /etc/conf.d/nfs
    permissions: '0644'
    content: |
      OPTS_RPC_MOUNTD=""
  - path: /etc/exports
    permissions: '0644'
    content: /exports/ *(rw,async,no_subtree_check,no_root_squash,fsid=0)
  - path: /exports/hello
    content: world
```

The first prepares an client config file ```/etc/conf.d/nfs``` for mounting nfs shares. The file ```/etc/exports``` defines, which folders will be exported by NFS. In this example just the folder ```/exports```. The third file ```/exports/hello``` is just there, to make sure the folder ```/exports``` is created and contains a simple file to test the client.

Next we need to add two services in the coreos / units section:

```
- name: rpc-statd.service
  command: start
  enable: true
- name: nfsd.service
  command: start
  enable: truestrong text
```

That's it. If you now recreate your Environment (```vagrant destroy``` && ```vagrant up```), there is a NFS share available on each host. Let's test it. Connect to any of the machines (```vagrant ssh core-03```). Then run the following tests:

```
$ showmount core-01
Hosts on core-01:
172.17.8.103
$ showmount -e core-01
Export list for core-01:
/exports *
$ sudo mount core-01:/exports /mnt
$ cat /mnt/hello
world
```

Everything seems to be okay, so let's move on and store our docker volumes there.

### Share your data with NFS
Since version 1.8 Docker does support Volume Driver Plugins. Such a Plugin can extend Docker with new volume drivers. For our purpose, we need a Plugin which supports Volumes on NFS shares. There are different plugins available. My decision was to use [Netshare by Containx](https://github.com/ContainX/docker-volume-netshare). Let's first try it manually. Connect to your host core-02 (```vagrant ssh core-02```) and run the following commands:

```
$ wget https://github.com/ContainX/docker-volume-netshare/releases/download/v0.18/docker-volume-netshare_0.18_linux_amd64-bin
docker-volume-netshare_0.18_linux_amd64-bin
Resolving github.com... 192.30.253.112
Connecting to github.com|192.30.253.112|:443... connected.
...
HTTP request sent, awaiting response... 200 OK
Length: 10917152 (10M) [application/octet-stream]
Saving to: 'docker-volume-netshare_0.18_linux_amd64-bin'

$ chmod ugo+x chmod ugo+x ./docker-volume-netshare_0.18_linux_amd64-bin
```
The Docker Volume Netshare Plugin is nothing else than an executable binary. Integration in Docker is as simple as running this binary and passing the wanted storage filesystem as argument (either nfs, efs or cifs). To enable NFS support in the local Docker daemon we execute the following command:

```
$ sudo ./docker-volume-netshare_0.18_linux_amd64-bin nfs&
[1] 4694
INFO[0000] == docker-volume-netshare :: Version: 0.18 - Built: 2016-05-27T20:14:07-07:00 ==
INFO[0000] Starting NFS Version 4 :: options: ''
```
Please be aware of the ampersand at the end (&). The binary doesn't start a daemon but runs synchronously. The ampersand tells your shell to start the process in the background. This way, your shell will not be blocked for the further tests:

```
$ docker run -i -t --volume-driver=nfs -v core-01/exports:/mount busybox /bin/sh
INFO[0054] Mounting NFS volume core-01:/exports on /var/lib/docker-volumes/netshare/nfs/core-01/exports

/# cat /mount/hello
world

/# echo "another hello" > /mount/greeting

/# exit
INFO[0255] Unmounting volume name core-01/exports from /var/lib/docker-volumes/netshare/nfs/core-01/exports
INFO[0255] Removing un-managed volume

$ docker run -i -t --volume-driver=nfs -v core-01/exports:/mount busybox cat /mount/greeting
INFO[0301] Mounting NFS volume core-01:/exports on /var/lib/docker-volumes/netshare/nfs/core-01/exports
another hello
INFO[0301] Unmounting volume name core-01/exports from /var/lib/docker-volumes/netshare/nfs/core-01/exports
INFO[0301] Removing un-managed volume
```

This looks promising. Getting the volume plugin to run was as easy as to load a binary and run it. Afterwards we can use the new volume-driver ```nfs``` to access our hello file. A newly created file is persisted over different runs.

Now let's modify our setup, so that the nfs volume plugin is loaded by default. To do so, create a new file ```bootstrap.sh``` in the folder of your Vagrantfile. Paste in the following content:

```
#!/usr/bin/env bash
mkdir /home/core/bin
wget -nv -O /home/core/bin/docker-volume-netshare https://github.com/ContainX/docker-volume-netshare/releases/download/v0.18/docker-volume-netshare_0.18_linux_amd64-bin
chmod ugo+x  /home/core/bin/docker-volume-netshare
```

Next we need to tell Vagrant to run this shell script when provisioning the machines. Add the following to your Vagrantfile (within to loop over the instances):

```
config.vm.provision :shell, path: "bootstrap.sh"
```

The last step is to run the Docker Volume Netshare plugin as a service on startup. We do this by defining the following new unit in our cloud-config (user-data file):

```
- name: docker-volume-netshare.service
  command: start
  enable: true
  content: |
    [Unit]
    Description=Enable Docker Volume Netshare Plugin for NFS
Requires=docker.service

    [Service]
    ExecStart=/home/core/bin/docker-volume-netshare nfs

    [Install]
    WantedBy=default.target
```

Now rebuild your environment (```vagrant reload --provision```).

### Running the tests

We can now run the tests from [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/). First let's create our named volume ```data```:

```
> docker volume create --driver nfs --name data --opt share=core-01:/exports
exports

> docker volume inspect data
[
    {
        "Name": "data",
        "Driver": "nfs",
        "Mountpoint": "/var/lib/docker-volumes/netshare/nfs/data",
        "Labels": {},
        "Scope": "local"
    }
]
```
The important difference here is using ```nfs``` as driver and the nfs share to use (core-01:/exports) as opt argument. Now let's run our tests with this named volume:


```
> docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 8
-rw-r--r--    1 root     root             5 Sep  7 09:41 hello

> docker run -i -t -v data:/my-shared-data busybox cp /etc/hostname /my-shared-data

> docker run -i -t -v data:/my-shared-data busybox ls -l /my-shared-data
total 16
-rw-r--r--    1 root     root             5 Sep  7 09:41 hello
-rw-r--r--    1 root     root            13 Sep  7 09:55 hostname

> docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
f78fd608ac88

> docker run -i -t -v data:/my-shared-data busybox cat /my-shared-data/hostname
f78fd608ac88
```

Looks fine. Everything runs like expected. But did the commands run on different hosts? Let's check:

```
E:\Projekte\coreos-vagrant>docker ps --all
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS            PORTS               NAMES
0ac66948dc37        busybox             "cat /my-shared-data/"   3 minutes ago       Exited (0) 3 minutes ago                       core-02/elated_knuth
00b69e2eaed6        busybox             "cat /my-shared-data/"   3 minutes ago       Exited (0) 3 minutes ago                       core-03/tiny_hawking
2dcb47f2b7ad        busybox             "ls -l /my-shared-dat"   3 minutes ago       Exited (0) 3 minutes ago                       core-01/furious_cray
f78fd608ac88        busybox             "cp /etc/hostname /my"   3 minutes ago       Exited (0) 3 minutes ago                       core-02/drunk_turing
bf2e1dd43439        busybox             "ls -l /my-shared-dat"   5 minutes ago       Exited (0) 5 minutes ago                       core-03/zen_pare
```

The names prefixes (core-01, core-02, core-03) show, that indeed all our hosts were used and could share the data. Problem solved.

## Lessons Learned
NFS seems to be a viable option for sharing data between docker containers running on different hosts. It does integrate nicely into Docker when using named volumes. You only need to pass the needed arguments when creating the named volume. All the other commands work like with locally stored named volumes. This should integrate well into the Docker ecosystem, most importantly docker-compose.
