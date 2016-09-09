---
layout: post
title:  Securing the Docker for CoreOS Vagrantfile template
---
The [coreos-vagrant](https://github.com/coreos/coreos-vagrant) Template is a minimal template to get a coreos cluster with Docker running. Docker is exposed via tcp/2375. This is an unsecured http connection. That is fine for trusted test environments. But for production systems and untrusted environments, it is necessary to secure this connection via transport layer security ([TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)). Docker supports TLS out of the box. We just need to create keys and change the configuration.

<!-- more -->

I assume, that you have an development machine with the following components installed:

- Vagrant (current version `1.8.5`)
- VirtualBox (current version `5.1.2`)
- Git client (current version `2.9.2`)
- Docker Toolbox (version `1.12.0`)

I further assume, that you have installed git with the optional Unix tools from the Windows Command Prompt option (when running on windows, as I do). Have a look here [Deploying a multi node Jenkins environment with docker and coreos - Part 1](/Master-Slave-Jenkins-With-Docker-Part1/) for further details.

### Step 1: Create necessary keys and certificates
First we need to create server and client keys and certificates. To do so, I follow the instructions from the official Docker docs [Protect the Docker daemon socket](https://docs.docker.com/engine/security/https/). To use this from windows, we can start a unix shell by typing ```bash``` in the command line (bash comes with the git installation). Now type in ```cd``` (without arguments) to change your working directory to your user home. Docker expects the client certificates in the directory .docker here. If it doesn't exist yet, run ```mkdir .docker```. Afterwards, change into this folder ```cd .docker```. I did prepare a little [helper script](https://raw.githubusercontent.com/ender74/secured-coreos-vagrant/master/create_keys.sh), which will create all necessary files. Be aware, that this script is intended to be run with the git-bash. If you use another shell, it might be necessary to replace //CN= with /CN.

```
#!/bin/bash

rm -f *.pem
rm -f *.csr

HOST="core-01"
IP="IP:172.17.8.101,IP:172.17.8.102,IP:172.17.8.103,IP:127.0.0.1"

openssl genrsa -aes256 -passout pass:docker -out ca-key.pem 4096

openssl req -subj "//CN=$HOST" -new -x509 -days 365 -key ca-key.pem -sha256 -passin pass:docker -out ca.pem

openssl genrsa -out server-key.pem 4096

openssl req -subj "//CN=$HOST" -sha256 -new -key server-key.pem -out server.csr

echo subjectAltName = $IP > extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -passin pass:docker -out server-cert.pem -extfile extfile.cnf

openssl genrsa -out key.pem 4096

openssl req -subj '//CN=client' -new -key key.pem -out client.csr

echo extendedKeyUsage = clientAuth > extfile.cnf

openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -passin pass:docker -out cert.pem -extfile extfile.cnf

chmod -v 0400 ca-key.pem key.pem server-key.pem

chmod -v 0444 ca.pem server-cert.pem cert.pem
```

### Step 2: Change configuration and start the cluster
To enable TLS for the Docker daemon, we need to copy the servers keys and certificates to the Docker host and modify the Docker options.

If you do not want to make the changes for yourself, grab an already modified version [here](https://github.com/ender74/secured-coreos-vagrant).

First open the Vagrantfile and make the following modifications:

At the beginning, right after CONFIG = File.join... insert the following lines:

```
DOCKER_PATH = File.join(Dir.home, ".docker")
KEYS_CA_PATH = File.join(DOCKER_PATH, "ca.pem")
KEYS_KEY_PATH = File.join(DOCKER_PATH, "server-key.pem")
KEYS_CERT_PATH = File.join(DOCKER_PATH, "server-cert.pem")
```
This defines three variables with the location of the needed keys and certificates. Those files need to be copied to the Virtual Machine at provisioning. To do so, add the following lines after the if File.exist?(CLOUD_CONFIG_PATH) ... end block (within the loop over all instances).

```
config.vm.provision :shell, :inline => "mkdir /etc/docker/", :privileged => true
if File.exist?(KEYS_CA_PATH)
  config.vm.provision :file, :source => "#{KEYS_CA_PATH}", :destination => "/tmp/ca.pem"
  config.vm.provision :shell, :inline => "mv /tmp/ca.pem /etc/docker/", :privileged => true
else
  puts "Could not find ca.pem with location: " + KEYS_CA_PATH
end
if File.exist?(KEYS_KEY_PATH)
  config.vm.provision :file, :source => "#{KEYS_KEY_PATH}", :destination => "/tmp/server-key.pem"
  config.vm.provision :shell, :inline => "mv /tmp/server-key.pem /etc/docker/", :privileged => true
else
  puts "Could not find server-key.pem with location: " + KEYS_KEY_PATH
end
if File.exist?(KEYS_CERT_PATH)
  config.vm.provision :file, :source => "#{KEYS_CERT_PATH}", :destination => "/tmp/server-cert.pem"
  config.vm.provision :shell, :inline => "mv /tmp/server-cert.pem /etc/docker/", :privileged => true
else
  puts "Could not find server-cert.pem with location: " + KEYS_CERT_PATH
end
```

This copies all files to /etc/docker/. The next step is to tell the docker daemon to use those files. To do so, modify your user-data file. You need to change the port for the docker-tcp.socket from 2375 to 2376. Then add the following block at the end of the file:

```
- name: docker.service
  drop-ins:
  - name: 10-docker-swarm.conf
    content: |
      [Service]
      Environment="DOCKER_OPTS=--tlsverify --tlscacert=/etc/docker/ca.pem --tlscert=/etc/docker/server-cert.pem --tlskey=/etc/docker/server-key.pem"
```

This defines a secured Docker Daemon on Port 2376. The drop-in sets the needed Docker Options for TLS authentication / verification. Voilà, that's it. Start your cluster with ```vagrant up```. When the cluster is started, run some tests:

```
> set DOCKER_HOST=172.17.8.101:2376

> set DOCKER_TLS_VERIFY=1

> docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
...
Name: core-01
ID: 7PQT:XIE4:F76D:CYOS:JJJN:5LPL:K2BY:2SLC:HYIV:24WQ:YK7P:PBJD
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Insecure Registries:
 127.0.0.0/8

> set DOCKER_HOST=172.17.8.102:2376

> docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
...
Name: core-02
ID: AIAJ:S7UT:WZ7K:GY3B:ARNN:P6FT:3NI5:4SYF:KXHE:SGWB:P2PQ:HB2I
Docker Root Dir: /var/lib/docker
Debug Mode (client): false
Debug Mode (server): false
Registry: https://index.docker.io/v1/
Insecure Registries:
 127.0.0.0/8
```

You may use curl from within the git bash to test the https connection:

```
$ export HOST=172.17.8.101
$ curl https://$HOST:2376/v1.24/info --insecure --cert ~/.docker/cert.pem --key ~/.docker/key.pem --cacert ~/.docker/ca.pem
{"ID":"7PQT:XIE4:F76D:CYOS:JJJN:5LPL:K2BY:2SLC:HYIV:24WQ:YK7P:PBJD","Containers":0,"ContainersRunnin
g":0,"ContainersPaused":0,"ContainersStopped":0, ... }
```

When reading [Protect the Docker daemon socket](https://docs.docker.com/engine/security/https/) you might wonder, why we didn't need to copy the client keys and certicates. This is the advantage of generating all keys in the ~/.docker directory. Docker looks there by default.

### Conclusion
We now have a secured API gateway for each individual Docker daemon within our cluster. For a production system, we might choose to disable the unsecured port 2375 completely. It's enough to remove the docker-tcp.socket Unit from your user-data file if you want to do so.
