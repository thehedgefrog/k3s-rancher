# Documentation <!-- omit in toc -->

This guide should be seen as a general reference, not a complete walkthrough.  It assumes you have a good understanding of Docker, Docker-Compose and running several Docker containers at once.  It also assumes you are well-versed in networking and understand the fundamental concepts.

### Table of contents <!-- omit in toc -->

- [Basic Considerations](#basic-considerations)
  - [Hardware](#hardware)
  - [Operating System](#operating-system)
  - [High Availability](#high-availability)
- [Why do you...?](#why-do-you)
  - [Why K3s](#why-k3s)
  - [Why Rancher](#why-rancher)
- [What I am using](#what-i-am-using)
  - [Hardware](#hardware-1)
  - [Resources](#resources)
- [Architecture](#architecture)

## Basic Considerations

### Hardware
K3s will run on surprisingly minimal hardware.  However for best results, you'll want at least 2GB of RAM and a 1.5GHz CPU.  While it is perfectly possible to replicate this guide on ARM, we are focusing on x64 here - besides, there are more tutorials and guides around touching on K8s on ARM.

### Operating System
We'll work with Ubuntu 18.04 LTS here, as it is the recommended OS by Rancher Labs to run K3s.  While it appears K3os is a purpose built OS for K3s, I could never have it work right (and trust me, I tried.)

### High Availability
I am replicating an HA configuration here - but it is **not** HA as there are several single points of failure.  The external database is the obvious one, but the simple fact of having your cluster at home, behind a single internet connection, and subject to power failures beyond the capacity of your UPS, means this is not true highly available.  However, this config is definitely mitigating risks of a single node failure bringing down your cluster, and lets you perform maintenance without downtime.

## Why do you...?

### Why K3s
It's stupid easy.  It really is.  With the k3sup (pronounced ketchup) script by Alex Ellis, deploying a K3s HA-style cluster takes, and that's a literal and measured figure, 120 seconds.  It's also light, doesn't take up much resources, and runs on pretty much everything.

Also, it perfectly integrates with Rancher - which brings us to...

### Why Rancher
OMG, it's a GUI, it's not the right way, you need to use the command line for everything else you're not truly learning Kubernetes...

I know.  I don't work with K8s, I don't need the skill for work, I'm doing this for fun.  As I work on this project, I'm learning the CLI way little by little.  Rancher just works.  It's easy to use and consolidates everything, and as I'm doing it on my free time, it's perfect for me.  Plus, let's be real, no business runs CLI only, they all use tools (many use Rancher, actually) and they almost never run it on-prem as we are.

## What I am using
### Hardware
I run various hardware, not all at the same time, depending on what I do.  The hardware in use for this cluster will definitely change as I go, and I will keep an up-to-date list [here](cluster-hardware.md).

### Resources
I have a MySQL database on the NAS, that I really hope to transition permanently out of its role in the cluster to have more of a semblance of HA.  The NAS is also the main NFS target for storage.

## Architecture
We'll use a 3 node configuration.  Two of those will be servers, one will be a worker node.  The servers are addressable, ie. they will get assigned workloads and also function as workers.  Since we're using an external database, the minimum amount of servers is 2, and the minimum amount of non-server worker nodes is 0.

We need an external SQL database, in our case it'll be a MariaDB MySQL database.  The database can't run within your cluster.

K3s is flexible, so you can add more servers and nodes as you grow.
