---
layout: post
title:  Deploying a multi node Jenkins environment with docker and coreos - Part 2
---

In the [first part of this series](/Master-Slave-Jenkins-With-Docker-Part1) we got a docker-swarm
running with vagrant and coreos. In this part we are going to use this docker-swarm to run a master-slave
[Jenkins](https://jenkins.io/) installation. Jenkins is a continous delivery tool. You can configure jobs to build your
projects and deploy them. There are plugins for many build environments available. Master-Slave setup means, that the
Jenkins master doesn't necessarily run all the build jobs. The master is there for running and orchestrating the jobs.
The build jobs themselves run on dedicated build slaves.

<!-- more -->
