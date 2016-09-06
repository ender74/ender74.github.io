---
layout: post
title:  Sharing Volumes in Docker with NFS
---

After my initial tests with [Sharing Volumes with docker-swarm](/Sharing-Volumes-With-Docker-Swarm/) based on Docker 1.11 I still had some open issues. This docker-swarm was no transparent replacement for a plain Docker host. The main problem was sharing data between containers on different swarm nodes. Some quick tests with the integrated swarm mode for Docker 1.12 showed, that this new mode takes an different approach than the docker-swarm containers before. You need to create special services to utilize it. It doesn't seem to solve the problem with sharing volumes between hosts though.
So I decided to try the most obvious solution next. Docker has no problem with networking. You can create networks to connect multiple containers. So it is possible to use a network file system like NFS to share data between containers.

### Running an NFS server
If you already have an NFS server, you may skip this part. For production it's probably best to use an external file service like NFS or Amazons EFS. But for some tests, it's easier to start an NFS service with our virtual Vagrant Environment. For this we will be editing the cloud-config (user-data file) as follows. First create some files:

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

Thats it. If you now recreate your Environment (```vagrant destroy``` && ```vagrant up```), there is a NFS mount available on each host. Let's test it. Connect to any of the machines (```vagrant ssh core-03```). Then run the following tests:

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
