# Documentation <!-- omit in toc -->

This guide should be seen as a general reference, not a complete walkthrough.  It assumes you have a good understanding of Docker, Docker-Compose and running several Docker containers at once.  It also assumes you are well-versed in networking and understand the fundamental concepts.

## Table of contents <!-- omit in toc -->

- [Basic Considerations](#basic-considerations)
  - [Hardware](#hardware)
  - [Operating System](#operating-system)
  - [High Availability](#high-availability)
  - [Domain name](#domain-name)
  - [DNS and network](#dns-and-network)
- [Why do you...?](#why-do-you)
  - [Why K3s](#why-k3s)
  - [Why Rancher](#why-rancher)
- [What I am using](#what-i-am-using)
  - [Hardware](#hardware-1)
  - [Resources](#resources)
- [Architecture](#architecture)
  - [A word on Rancher Labs' recommendations](#a-word-on-rancher-labs-recommendations)
- [Installing Linux](#installing-linux)
- [Setting up your MySQL database](#setting-up-your-mysql-database)
- [Installing K3s](#installing-k3s)
  - [Server install](#server-install)
  - [Node install](#node-install)
- [Getting started with kubectl](#getting-started-with-kubectl)
- [Installing Rancher](#installing-rancher)
  - [Helm](#helm)
  - [Cert-manager](#cert-manager)
  - [Rancher](#rancher)

# Basic Considerations

## Hardware
K3s will run on surprisingly minimal hardware.  However for best results, you'll want at least 2GB of RAM and a 1.5GHz CPU.  While it is perfectly possible to replicate this guide on ARM, we are focusing on x64 here - besides, there are more tutorials and guides around touching on K8s on ARM.

## Operating System
We'll work with Ubuntu 18.04 LTS here, as it is the recommended OS by Rancher Labs to run K3s.  While it appears K3os is a purpose built OS for K3s, I could never have it work right (and trust me, I tried.)

## High Availability
I am replicating an HA configuration here - but it is **not** HA as there are several single points of failure.  The external database is the obvious one, but the simple fact of having your cluster at home, behind a single internet connection, and subject to power failures beyond the capacity of your UPS, means this is not true highly available.  However, this config is definitely mitigating risks of a single node failure bringing down your cluster, and lets you perform maintenance without downtime.

## Domain name
You'll need a domain name for this project.  If you need to purchase one, I recommend [Namecheap](https://namecheap.pxf.io/7jan3).  While this is an affiliate link, and I greatly appreciate people using it, I still recommend them if you'd rather avoid my link.

You'll also need a DNS provider, and for that I recommend [CloudFlare](https://www.cloudflare.com/) (not an affiliate link).  While there are many DNS services that are compatible with dynamic DNS, I've had best results with CloudFlare (always on the free plan) and always go back to it for all my domain names.

## DNS and network
Unless you have a static IP, you will need to setup dynamic DNS, so that your main A record points to your home IP.  You'll also need to forward ports 80 and 443 to your cluster.

# Why do you...?

## Why K3s
It's stupid easy.  It really is.  With the k3sup (pronounced ketchup) script by Alex Ellis, deploying a K3s HA-style cluster takes, and that's a literal and measured figure, 120 seconds.  It's also light, doesn't take up much resources, and runs on pretty much everything.

Also, it perfectly integrates with Rancher - which brings us to...

## Why Rancher
OMG, it's a GUI, it's not the right way, you need to use the command line for everything else you're not truly learning Kubernetes...

I know.  I don't work with K8s, I don't need the skill for work, I'm doing this for fun.  As I work on this project, I'm learning the CLI way little by little.  Rancher just works.  It's easy to use and consolidates everything, and as I'm doing it on my free time, it's perfect for me.  Plus, let's be real, no business runs CLI only, they all use tools (many use Rancher, actually) and they almost never run it on-prem as we are.

# What I am using
## Hardware
I run various hardware, not all at the same time, depending on what I do.  The hardware in use for this cluster will definitely change as I go, and I will keep an up-to-date list [here](cluster-hardware.md).

## Resources
I have a MySQL database on the NAS, that I really hope to transition permanently out of its role in the cluster to have more of a semblance of HA.  The NAS is also the main NFS target for storage.

# Architecture
We'll use a 3 node configuration.  Two of those will be servers, one will be a worker node.  The servers are addressable, ie. they will get assigned workloads and also function as workers.  Since we're using an external database, the minimum amount of servers is 2, and the minimum amount of non-server worker nodes is 0.  We need an external SQL database, in our case it'll be a MariaDB MySQL database.  The database can't run within your cluster.

K3s is flexible, so you can add more servers and nodes as you grow.  If you don't need HA, it is perfectly fine to run a cluster of several VMs on a single host.  Actually, it's also perfectly fine to run your entire cluster on a single host, however you're losing out on the benefits of scale.

What's important to understand is that your *servers* are the important pieces.  If all servers go down, your entire cluster goes down with it.  So it's always better to maintain a balance between your servers and your worker nodes.  If you have 12 machines, don't have 2 servers and 10 worker nodes, have 4 servers and 8 workers.  However, contrary to popular belief, your servers don't need to be the most powerful machines in your cluster.

## A word on Rancher Labs' recommendations
Rancher Labs recommends having a separate cluster to run Rancher itself, and then have *another* cluster to run your workloads.  While that makes complete sense in a business production environment, at home I don't feel like it's worth it, and consequently if you follow my steps you'll end up with one cluster for everything.  Please feel free to do as you please, making more clusters isn't more difficult.

# Installing Linux
You'll want to get started by installing Ubuntu 18.04 LTS Server on all your machines.  I haven't tested with other versions (other than K3os which ended up being terrible), so feel free to install another distribution or version but YMMV.

At this step, I usually make sure to setup my hostname, static IP and VLAN tag according to whatever addressing convention I'm using for the project, install OpenSSH and retrieve my public key from GitHub.  Make sure to disable password authentication for SSH for maximum security.  You can safely stay with just that, complete the install, and reboot.

# Setting up your MySQL database
You'll need a DB dedicated to K3s, it'll get populated by quite a bit of data (~20MB, with another 10MB of overhead).  As a security measure, I always create a user that gets permissions only on that one database and generate a secure password.  Make note of that password, we'll use it in a few minutes.

K3s also supports PostgreSQL but I won't be using it, you shouldn't have trouble adapting the steps below for it if you choose to use it.

# Installing K3s
This is the magic step.  We'll use the fantastic [k3sup utility](https://github.com/alexellis/k3sup) by Alex Ellis, which will make this part, well, almost magic.  It'll use SSH to connect to your machines, so make sure your public keys are loaded in your targets, and your private key is in your ~/.ssh/ directory and named *id_rsa*.  If your key has a passcode (it should!), the script will call for you to enter it.

**You'll want to complete these steps from your main computer, not from your K3s machines**.

First of all, let's install k3sup using [these steps](https://github.com/alexellis/k3sup/blob/master/README.md#download-k3sup-tldr).  Test your installation by typing `k3sup`, you should see the project logo and the basic commands.

## Server install
At this point, we'll build our commands that will install K3s on your two servers.  It'll consist of a few things:

`k3sup install` is the command that'll setup the target as a server (also known as control plane).

`--user` will have your username on the target machine.  I always use **cow** as a username for Rancher machines (get it?) and **k3s-xxx** as a hostname, with s01... for servers and n01... for worker nodes, but you do whatever you want.

`--ip` will have your target machine IP.  I'll use 10.25.150.1x for the servers and 10.25.150.2x for the workers, again you do you.

`--datastore` will have your MySQL connection string.  It needs to be fiddled with a little, so let's do that first.  You'll want to do `mysql://username:password@tcp(your.mysql.ip.address:port)/databasename`.

...and that's it, it'll setup your first machine as your K3s server #1.  Here's what my connection string looks like (and even though you can't access my DB from the public internet, yes it's a fake password).
```sh
k3sup install --user cow --ip 10.25.150.11 --datastore="mysql://k3s:a1b2c3d4e5f6@tcp(10.44.121.1:3306)/k3scluster
```

You'll see that a *kubeconfig* file will have been created on your desktop or in your ~/ directory, save it under another name and/or elsewhere as you'll need it later to use `kubectl`.

Then you repeat the command for any other server(s).  I'm using two, so I'll just repeat the command and change the target IP to 10.25.150.12.  It'll generate a new *kubeconfig* file each time, but the first one will link all the others so don't worry about it.

```sh
k3sup install --user cow --ip 10.25.150.12 --datastore="mysql://k3s:a1b2c3d4e5f6@tcp(10.44.121.1:3306)/k3scluster
```

Repeat it as many times as necessary for your number of servers.  Voilà!  You have a K3s cluster.  Celebrate with a beer, a coffee, a Kombucha, whatever you enjoy drinking.

## Node install
It's now the time to install K3s on your worker nodes.

`k3sup join` will be our command this time.

`--user`, once again, your username on the target machine.  I'm still using **cow** as there is no need to have different usernames across your cluster.

`--server-ip` should be self-explanatory.  You can use either of your server IPs.

`--ip` is your target worker's IP, within my cluster it'll be 10.25.150.21.

That's it!
```sh
k3sup join --user cow --server-ip 10.25.150.11 --ip 10.25.150.21
```

You're done!  Take another sip of your beverage of choice.

# Getting started with kubectl
We'll be using `kubectl` extensively within our cluster.  Once again, you'll be using it from your main machine, not your servers.  It works fine on Windows with either PowerShell or WSL, and obviously on Linux and Mac.  Please refer to [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for steps on how to install it.

Once it's installed, you'll need to copy the info in the *kubeconfig* file we saved earlier into the kubeconfig file of your new install (by default, you'll find it at `~/.kube/config`).  That'll point your local install of kubectl to our brand new cluster.  We're now ready to test!

Type `kubectl get nodes -o wide` and you should see the three machines of your cluster.  Awesome!  If you only want a Kubernetes cluster, feel free to stop right here.  If you want to go ahead with installing Rancher, keep going!

# Installing Rancher
Rancher is a WebUi for Kubernetes.  We'll be able to do most of the work we'll do on our cluster from it, but not all of it.  However, I personally like it more than the Google K8s dashboard.

The info here is in support of, not in replacement of the [official Rancher documentation](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/helm-rancher/) so you'll want to have it open in another tab.

## Helm
Helm is kind of the K8s version of  Docker Compose, if I had to compare.  It lets you install applications that depend on others within your Kubernetes environment.  Like most things, you'll install it on your workstation, not your target machines.  Follow [the instructions](https://helm.sh/docs/intro/install/) to install Helm on your OS of choice.

You can now proceed to steps 2 and 3 of the Rancher docs mentioned earlier.  I recommend using the Stable repo of Rancher, but feel free to use the Latest if you're feeling adventurous.  **Don't use the Alpha repo, even for home use.**

## Cert-manager
While you can use self-signed certs, I recommend using Let's Encrypt to generate a certificate for your Rancher application.  To do so, you'll want to follow the instructions under step 5 of the Rancher docs, and click Expand to get the commands.  I won't post them here as the recommended cert-manager version could change at any time.

## Rancher
It's now time to install Rancher itself.  Under step 6, select the Let's Encrypt tab, then follow the steps.  In less than 5 minutes, Rancher will be up and running.  Confirm with step 7.