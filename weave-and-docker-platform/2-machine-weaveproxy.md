---
layout: guides
title: "2. Using Weave with Docker Machine via proxy"
permalink: /guides/weave-and-docker-platform/chapter2/machine-with-weave-proxy.html
tags: docker, machine, cli, virtualbox, dns, ipam, proxy, hello-weave-app
---

> ### ***Creating distributed applications with Weave and the Docker platform***
>
> - Chapter 1: [Using Weave with Docker Machine][ch1]
> - Chapter 3: [Using Weave with Docker Machine and Swarm][ch3]
> - Chapter 4: [Creating and scaling multi-host Docker deployment with Swarm and Compose using Weave][ch4]

Weave allows you to focus on developing your application, rather than your infrastructure, and it works great with tools
like [Docker Machine](https://docs.docker.com/machine/). Here you will learn how to get started, you can then proceed to
a more advanced setup with Swarm and later Compose in following chapters of this guide.

## What you will build

[Docker Machine](https://docs.docker.com/machine/) makes it really easy to create Docker hosts (VMs) on your computer, on
cloud providers and inside your own data center. It creates servers, installs Docker on them, then configures the Docker
client to talk to them.

This chapter continues from [the previous one][ch1], which I encourage you to read first.

Here, we will look at how [Weave proxy][proxy] enables you to use Docker client, i.e. `docker run`, instead of `weave run`.
After you are familiar with the concept of proxy, we can proceed to the [next chapter][ch3] where we will introduce Swarm.

Through a few simple steps you will setup Weave on a single VirtualBox VM, it is really quite simple. You will deploy a
basic _"Hello, Weave!"_ application and then use WeaveDNS to access it from another Weave-attached Docker container.
This chapter uses very simple UNIX tools, hence no programming skills are required. Unlike we did in the [previous
chapter][ch1], we will use `docker` command direcly insted of `weave` command, simplifying the flow.

## What you will use

  - [Weave](http://weave.works)
  - [Docker & Machine](http://docker.com)

## What you will need to complete this chapter

  - 10-15 minutes
  - [`docker-machine`](http://docs.docker.com/machine/#installation) binary (_`>= 0.2.0`_)
  - [`docker`](https://docs.docker.com/installation/#installation) binary, at lest the client (_`>= v1.6.x`_)
  - [VirtualBox](https://www.virtualbox.org/wiki/Downloads) (_`>= 4.3.x`_)
  - `curl` (_any version_)

_If you have followed through [the previous chapter][ch1], you should already have all of these dependencies installed._

If you are using OS X, then you can install these tools with Homebrew like this:

    brew install docker docker-machine

For other operating systems, please refer to links above.

If you haven't yet installed VirtualBox, be sure to follow [installation instructions for your OS](https://www.virtualbox.org/wiki/Downloads).

## Let's go!

### Launch

If you haven't followed through the [previous chapter][prev1], you can run the following commands to get a VM with Weave
network setup. You also should run these if you chose to destroy the `weave-1` machine.

    curl -OL git.io/weave
    chmod +x ./weave
    docker-machine create -d virtualbox weave-1
    export DOCKER_CLIENT_ARGS=$(docker-machine config weave-1)
    ./weave launch
    ./weave launch-dns 10.53.1.1/16

Next, before we can launch Weave proxy, we will need to obtain the TLS settings from Docker daemon

    tlsargs=$(docker-machine ssh weave-1 \
      "cat /proc/\$(pgrep /usr/local/bin/docker)/cmdline | tr '\0' '\n' | grep ^--tls | tr '\n' ' '")

Now we can launch the proxy with

    ./weave launch-proxy --with-dns --with-ipam $tlsargs

If you wish to find out more about how Weave proxy works, you should [read the documentation][proxy]. Here, all we need
to know is that it can be accessed on port 12375 and we need to point docker client at it next.

Let's modify the `DOCKER_CLIENT_ARGS` environment variable

    export DOCKER_CLIENT_ARGS=$(docker-machine config weave-1 | sed 's|:2376|:12375|')

and test if all is well with

    docker $DOCKER_CLIENT_ARGS info

### Deploy

All tricks are done, and we can start our _"Hello, Weave!"_ app using Docker client instead of Weave script like this

    > docker $DOCKER_CLIENT_ARGS run -d --name=pingme gliderlabs/alpine nc -p 4000 -lk -e echo 'Hello, Weave!'
    635ebf591cedc01610faea18b78ff977830dfc5bcb239accc833243304da619d

This is a simple netcat (aka `nc`) server that runs on TCP port 4000 and sends a short `Hello, Weave!` message to each
client that connects to it.

Next, we can start an interactive test container without having to call `docker attach` like we had to do previously.

    > docker $DOCKER_CLIENT_ARGS run --name=pinger -ti gliderlabs/alpine sh -l

And now let's repeat the test just as we [did before][prev2].

Ping the other container by it's DNS name

    pinger:/# ping -c3 pingme.weave.local
    PING pingme.weave.local (10.128.0.1): 56 data bytes
    64 bytes from 10.128.0.1: seq=0 ttl=64 time=0.100 ms
    64 bytes from 10.128.0.1: seq=1 ttl=64 time=0.114 ms
    64 bytes from 10.128.0.1: seq=2 ttl=64 time=0.111 ms

    --- pingme.weave.local ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.100/0.108/0.114 ms

Test if it responds on TCP port 4000 as expected

    pinger:/# echo "What's up?" | nc pingme.weave.local 4000
    Hello, Weave!

We can exit the test container now.

    pinger:/# exit

And, as all worked well, let's get rid of both containers by running

    > docker $DOCKER_CLIENT_ARGS rm -f pingme pinger
    pingme
    pinger

## Cleanup

In order to proceed to the [next chapter][ch3] you should remove the `weave-1` VM we used here.

    docker-machine rm -f weave-1

## Summary

In this short chapter we have learned how to use Weave with Docker Machine and deployed a simple _"Hello, Weave!"_ service.
Most importantly, you now shuld be familiar with all the commands you need to use in order to create a VM and start containers
on it remotely as well as have an understanding of how to integrate Weave proxy, which allows you to use `docker` command
directly. Now you can proceed to the next step and look at how to setup more then one VM, letting Docker Swarm to schedule
containers with [Weave Net](/net) providing transparent connectivity across multiple Docker hosts.

[proxy]: http://docs.weave.works/weave/latest_release/proxy.html
[prev1]: /guides/weave-and-docker-platform/chapter1/machine.html#launch
[prev2]: /guides/weave-and-docker-platform/chapter1/machine.html#deploy
[ch1]: /guides/weave-and-docker-platform/chapter1/machine.html
[ch2]: /guides/weave-and-docker-platform/chapter2/machine-with-weave-proxy.html
[ch3]: /guides/weave-and-docker-platform/chapter3/machine-and-swarm-with-weave-proxy.html
[ch4]: /guides/weave-and-docker-platform/chapter4/compose-scalable-swarm-cluster-with-weave.html
