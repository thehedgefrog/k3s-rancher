# Documentation

This guide should be seen as a general reference, not a complete walkthrough.  It assumes you have a good understanding of Docker, Docker-Compose and running several Docker containers at once.  It also assumes you are well-versed in networking and understand the fundamental concepts.

*table of contents here*

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