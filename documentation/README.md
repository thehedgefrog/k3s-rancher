# Documentation <!-- omit in toc -->

This guide should be seen as a general reference, not a complete walkthrough.  It assumes you have a good understanding of Docker, Docker-Compose and running several Docker containers at once.  It also assumes you are well-versed in networking and understand the fundamental concepts.

## Table of contents <!-- omit in toc -->

- [1. Basic Considerations](#1-basic-considerations)
  - [1.1. Hardware](#11-hardware)
  - [1.2. Operating System](#12-operating-system)
  - [1.3. High Availability](#13-high-availability)
  - [1.4. Domain Name](#14-domain-name)
  - [1.5. DNS and Network](#15-dns-and-network)
- [2. Why do you...?](#2-why-do-you)
  - [2.1. Why K3s](#21-why-k3s)
  - [2.2. Why Rancher](#22-why-rancher)
- [3. What I am using](#3-what-i-am-using)
  - [3.1. Hardware](#31-hardware)
  - [3.2. Resources](#32-resources)
- [4. Architecture](#4-architecture)
  - [4.1. A word on Rancher Labs' recommendations](#41-a-word-on-rancher-labs-recommendations)
- [5. Installing Linux](#5-installing-linux)
- [6. Setting up your MySQL database](#6-setting-up-your-mysql-database)
- [7. Installing K3s](#7-installing-k3s)
  - [7.1. Server install](#71-server-install)
  - [7.2. Node install](#72-node-install)
- [8. Getting started with kubectl](#8-getting-started-with-kubectl)
- [9. Installing Rancher](#9-installing-rancher)
  - [9.1. Helm](#91-helm)
  - [9.2. Cert-manager](#92-cert-manager)
  - [9.3. Rancher](#93-rancher)
- [10. Using Rancher](#10-using-rancher)
  - [10.1. Setting up](#101-setting-up)
  - [10.2. Clusters, projects and namespaces](#102-clusters-projects-and-namespaces)
  - [10.3. Workloads, secrets, configmaps and ingresses](#103-workloads-secrets-configmaps-and-ingresses)
  - [10.4. Learning by doing - our first deployment](#104-learning-by-doing---our-first-deployment)
    - [10.4.1. Docker Compose](#1041-docker-compose)
    - [10.4.2. Rancher Deploy](#1042-rancher-deploy)
    - [10.4.3. Ingress](#1043-ingress)
  - [10.5. Certificates](#105-certificates)
    - [10.5.1. Staging](#1051-staging)
    - [10.5.2. Production](#1052-production)
  - [10.6. Persistent Storage](#106-persistent-storage)
  - [10.7. What's Next?](#107-whats-next)
- [11. Securing our cluster](#11-securing-our-cluster)
  - [11.1. Why?](#111-why)
  - [11.2. How?](#112-how)
  - [11.3. Setting up Authelia](#113-setting-up-authelia)
    - [11.3.1. Architecture](#1131-architecture)
    - [11.3.2. The Configuration File](#1132-the-configuration-file)
    - [11.3.3. Installing Authelia in Rancher](#1133-installing-authelia-in-rancher)
    - [Putting services behind Authelia](#putting-services-behind-authelia)
      - [Traefik Configuration](#traefik-configuration)
      - [Ingress Configuration](#ingress-configuration)
- [Reverse Proxying](#reverse-proxying)
  - [Create the Service](#create-the-service)
  - [Create the Ingress](#create-the-ingress)
- [Conclusion](#conclusion)
  - [How can I help?](#how-can-i-help)

# 1. Basic Considerations

## 1.1. Hardware
K3s will run on surprisingly minimal hardware.  However for best results, you'll want at least 2GB of RAM and a 1.5GHz CPU.  While it is perfectly possible to replicate this guide on ARM, we are focusing on x64 here - besides, there are more tutorials and guides around touching on K8s on ARM.

## 1.2. Operating System
We'll work with Ubuntu 18.04 LTS here, as it is the recommended OS by Rancher Labs to run K3s.  While it appears K3os is a purpose built OS for K3s, I could never have it work right (and trust me, I tried.)

## 1.3. High Availability
I am replicating an HA configuration here - but it is **not** HA as there are several single points of failure.  The external database is the obvious one, but the simple fact of having your cluster at home, behind a single internet connection, and subject to power failures beyond the capacity of your UPS, means this is not true highly available.  However, this config is definitely mitigating risks of a single node failure bringing down your cluster, and lets you perform maintenance without downtime.

## 1.4. Domain Name
You'll need a domain name for this project.  If you need to purchase one, I recommend [Namecheap](https://namecheap.pxf.io/7jan3).  While this is an affiliate link, and I greatly appreciate people using it, I still recommend them if you'd rather avoid my link.

You'll also need a DNS provider, and for that I recommend [Cloudflare](https://www.cloudflare.com/) (not an affiliate link).  While there are many DNS services that are compatible with dynamic DNS, I've had best results with Cloudflare (always on the free plan) and always go back to it for all my domain names.

## 1.5. DNS and Network
Unless you have a static IP, you will need to setup dynamic DNS, so that your main A record points to your home IP.  You'll also need to forward ports 80 and 443 to your cluster.

# 2. Why do you...?

## 2.1. Why K3s
It's stupid easy.  It really is.  With the k3sup (pronounced ketchup) script by Alex Ellis, deploying a K3s HA-style cluster takes, and that's a literal and measured figure, 120 seconds.  It's also light, doesn't take up much resources, and runs on pretty much everything.

Also, it perfectly integrates with Rancher - which brings us to...

## 2.2. Why Rancher
OMG, it's a GUI, it's not the right way, you need to use the command line for everything else you're not truly learning Kubernetes...

I know.  I don't work with K8s, I don't need the skill for work, I'm doing this for fun.  As I work on this project, I'm learning the CLI way little by little.  Rancher just works.  It's easy to use and consolidates everything, and as I'm doing it on my free time, it's perfect for me.  Plus, let's be real, no business runs CLI only, they all use tools (many use Rancher, actually) and they almost never run it on-prem as we are.

# 3. What I am using
## 3.1. Hardware
I run various hardware, not all at the same time, depending on what I do.  The hardware in use for this cluster will definitely change as I go, and I will keep an up-to-date list [here](cluster-hardware.md).

## 3.2. Resources
I have a MySQL database on the NAS, that I really hope to transition permanently out of its role in the cluster to have more of a semblance of HA.  The NAS is also the main NFS target for storage.

# 4. Architecture
We'll use a 3 node configuration.  Two of those will be servers, one will be a worker node.  The servers are addressable, ie. they will get assigned workloads and also function as workers.  Since we're using an external database, the minimum amount of servers is 2, and the minimum amount of non-server worker nodes is 0.  We need an external SQL database, in our case it'll be a MariaDB MySQL database.  The database can't run within your cluster.

K3s is flexible, so you can add more servers and nodes as you grow.  If you don't need HA, it is perfectly fine to run a cluster of several VMs on a single host.  Actually, it's also perfectly fine to run your entire cluster on a single host, however you're losing out on the benefits of scale.

What's important to understand is that your *servers* are the important pieces.  If all servers go down, your entire cluster goes down with it.  So it's always better to maintain a balance between your servers and your worker nodes.  If you have 12 machines, don't have 2 servers and 10 worker nodes, have 4 servers and 8 workers.  However, contrary to popular belief, your servers don't need to be the most powerful machines in your cluster.

## 4.1. A word on Rancher Labs' recommendations
Rancher Labs recommends having a separate cluster to run Rancher itself, and then have *another* cluster to run your workloads.  While that makes complete sense in a business production environment, at home I don't feel like it's worth it, and consequently if you follow my steps you'll end up with one cluster for everything.  Please feel free to do as you please, making more clusters isn't more difficult.

# 5. Installing Linux
You'll want to get started by installing Ubuntu 18.04 LTS Server on all your machines.  I haven't tested with other versions (other than K3os which ended up being terrible), so feel free to install another distribution or version but YMMV.

At this step, I usually make sure to setup my hostname, static IP and VLAN tag according to whatever addressing convention I'm using for the project, install OpenSSH and retrieve my public key from GitHub.  Make sure to disable password authentication for SSH for maximum security.  You can safely stay with just that, complete the install, and reboot.

# 6. Setting up your MySQL database
You'll need a DB dedicated to K3s, it'll get populated by quite a bit of data (~20MB, with another 10MB of overhead).  As a security measure, I always create a user that gets permissions only on that one database and generate a secure password.  Make note of that password, we'll use it in a few minutes.

K3s also supports PostgreSQL but I won't be using it, you shouldn't have trouble adapting the steps below for it if you choose to use it.

# 7. Installing K3s
This is the magic step.  We'll use the fantastic [k3sup utility](https://github.com/alexellis/k3sup) by Alex Ellis, which will make this part, well, almost magic.  It'll use SSH to connect to your machines, so make sure your public keys are loaded in your targets, and your private key is in your ~/.ssh/ directory and named *id_rsa*.  If your key has a passcode (it should!), the script will call for you to enter it.

**You'll want to complete these steps from your main computer, not from your K3s machines**.

First of all, let's install k3sup using [these steps](https://github.com/alexellis/k3sup/blob/master/README.md#download-k3sup-tldr).  Test your installation by typing `k3sup`, you should see the project logo and the basic commands.

## 7.1. Server install
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

## 7.2. Node install
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

# 8. Getting started with kubectl
We'll be using `kubectl` extensively within our cluster.  Once again, you'll be using it from your main machine, not your servers.  It works fine on Windows with either PowerShell or WSL, and obviously on Linux and Mac.  Please refer to [this guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for steps on how to install it.

Once it's installed, you'll need to copy the info in the *kubeconfig* file we saved earlier into the kubeconfig file of your new install (by default, you'll find it at `~/.kube/config`).  That'll point your local install of kubectl to our brand new cluster.  We're now ready to test!

Type `kubectl get nodes -o wide` and you should see the three machines of your cluster.  Awesome!  If you only want a Kubernetes cluster, feel free to stop right here.  If you want to go ahead with installing Rancher, keep going!

# 9. Installing Rancher
Rancher is a WebUi for Kubernetes.  We'll be able to do most of the work we'll do on our cluster from it, but not all of it.  However, I personally like it more than the Google K8s dashboard.

The info here is in support of, not in replacement of the [official Rancher documentation](https://rancher.com/docs/rancher/v2.x/en/installation/k8s-install/helm-rancher/) so you'll want to have it open in another tab.

## 9.1. Helm
Helm is kind of the K8s version of  Docker Compose, if I had to compare.  It lets you install applications that depend on others within your Kubernetes environment.  Like most things, you'll install it on your workstation, not your target machines.  Follow [the instructions](https://helm.sh/docs/intro/install/) to install Helm on your OS of choice.

You can now proceed to steps 2 and 3 of the Rancher docs mentioned earlier.  I recommend using the Stable repo of Rancher, but feel free to use the Latest if you're feeling adventurous.  **Don't use the Alpha repo, even for home use.**

## 9.2. Cert-manager
While you can use self-signed certs, I recommend using Let's Encrypt to generate a certificate for your Rancher application.  To do so, you'll want to follow the instructions under step 5 of the Rancher docs, and click Expand to get the commands.  I won't post them here as the recommended cert-manager version could change at any time.

## 9.3. Rancher
It's now time to install Rancher itself.  Under step 6, select the Let's Encrypt tab, then follow the steps.  In less than five minutes, Rancher will be up and running.  Confirm with step 7.

Another sip!

# 10. Using Rancher
## 10.1. Setting up
You now have a fully functional K3s cluster running Rancher.  When you followed the install instructions for Rancher, you've selected your portal domain name, let's assume `https://rancher.mydomain.com`.  You'll want to go to that page to get the intro screen.

First off, we'll make sure we got our SSL cert from Let's Encrypt by confirming the secure connection in your browser.  Then, select a secure passowrd. We'll make it more secure later, but you still want a secure pass - remember, right now this page is exposed to the whole internet.

## 10.2. Clusters, projects and namespaces
While clusters and namespaces are important all across the Kubernetes world, projects are something Rancher does specifically.  Remember that this is an enterprise product, so some things are way overkill in a home environment.  In short, projects are done so in an enterprise environment, you can limit access of users to specific applications within your cluster.

We are working with a single cluster, so that's as simple as it gets.  By default, it'll be called *local*.  Rancher will create two projects, Default and System.  I like to keep a separation between them, so that you run everything your cluster *depends* on within System, and everything else within Default (you can rename it!).

As for namespaces, these are actually very much everywhere when it comes to K8s.  They are an easy way to visually separate your types of apps in Rancher, manage secrets by making them available within a single namespace, and allocate resources to one or several deployments.  You're free to have as many, or as little, as you want - you'll find how you'd prefer to run it.  Personally, I create a namespace for every app or group of apps working together.  For example, I have a *download* namespace running Sonarr, Radarr, Jackett and DelugeVPN, and it's all dedicated to gathering my **collection of Linux ISOs.**

## 10.3. Workloads, secrets, configmaps and ingresses
These are standard things in K8s.  A *workload* is made of one or many *pods* part of a *deployment*.  You can deploy one or many workloads at a time, of one to many pods at a time.  A pod is known in Docker as a container.  You can run three times the same pod in a workload, so that you have total redundancy if you lose, for example, one server and one node, or to load balance within your three pods.

The beauty of Kubernetes is that since you have persistent storage, you can have only one pod in your workload, and if your node goes down, K8s will instantly know and relaunch it on another node, it'll be back up within seconds.  In a home environment, safe to say it'll be barely noticeable if at all.

*Secrets* are passwords, things you don't want public, and SSL certificates.  In Docker Swarm, you could put those secrets in files and reference them in your Compose file, in Kubernetes we'll put them in K8s secrets.

*Configmaps* are YAML files used to configure your applications.  When, in Docker, you had to put your config in a YAML file, and then SSH to your server to put it in a specific directory, here you'll create a configmap and mount it - usually as a volume.

*Ingresses* are basically the route from when traffic hits ports 80/443 to your workload.  K3s runs Traefik v1.7 as an ingress - I personally like it way more than the standard nginx, since it's significantly easier to configure and use.  I'm not sure why they don't use Traefik v2, but the diffrences are minor for what we'll be doing.  It's also super easy to set up reverse proxying to access services outside of your cluster using subdomains.

## 10.4. Learning by doing - our first deployment
Let's get a demo workload working.  Adrian Goins from Rancher Labs has created a demo app for Rancher that's perfect to try out by doing.

### 10.4.1. Docker Compose
Let's start by doing a Docker-Compose file.  We won't use it, but it's a perfect example of how you can translate a Compose file to Kubernetes.  Some people swear by Kompose, which does that translation for you, but it makes the unavoidable issues very hard to troubleshoot.  I'm basing it on my previous Docker stack, you could quite easily adapt it to your stack and run it on Docker first to compare.

It's based on Traefik 2.x, which I was using with Docker, while K3s is on 1.7, and I was assuming a domain name of `https://cows.mydomain.com` (you'll get it when you run the app).

```yaml
version: "3.7"

networks:
  traefik_net:
    external:
      name: traefik-net

services:

  rancher-demo:
    container_name: rancher_demo
    image: monachus/rancher-demo:latest
    restart: unless-stopped
    networks:
      - traefik-net
    ports:
      - "8080:8080"
    environment:
      - COW_COLOR=teal
      - TITLE="Trying Rancher!"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.rancher-demo.entrypoints=https"
      - "traefik.http.routers.rancher-demo.rule=HostHeader(`cows.mydomain.com`)"
      - "traefik.http.routers.rancher-demo.service=rancher-demo"
      - "traefik.http.services.rancher-demo.loadbalancer.server.port=8080"
```

With that, we have a container template.  What I could do (and quite frankly, what you should **very soon** take a few minutes to learn) is to deploy this very app on K8s from YAML and `kubectl` but there are excellent tutorials for that.  I'll focus on Rancher, and thus we'll deploy the same but on 3 pods, one on each of our nodes.

### 10.4.2. Rancher Deploy
So let's head to the *Default* project (click *local* or *Global* in the navbar, depending where in Rancher you are, then within the menu grab the *Global* project).  You should be in *Workloads* in the sub-navbar, if you aren't, head there.  By default, it'll be empty.  Let's click *Deploy* in the upper right corner.

We'll name our workload `rancher-demo`, make it a scalable deployment of 3 pods, use the `monachus/rancher-demo:latest` Docker image, and let's create a new namespace called `demo` by clicking *Add to a new namespace* above the Namespace box.  Remember when we installed Rancher?  You can also create a namespace with the `kubectl create namespace *name*` command.

Now, let's do Add Port, and publish 8080.  We'll make it a Cluster IP (internal only) as we'll be creating an ingress for it later.

Under Environment Variables, we'll add our two variables from our Compose file.  You don't need to put any brackets as Rancher takes care of it, so just fill the Variable boxes with `COW-COLOR` and `TITLE`, and the Value boxes with `teal` and `Trying Rancher!`.

You can safely head directly to Launch and give a few seconds for your pods to be up.

### 10.4.3. Ingress
Once you have your 3 pods running, head to *Load Balancing* on your sub-navbar.  We'll create our ingress here.  Click *Add Ingress* in the upper right corner.

We'll name it `rancher-demo` and add it to the same `demo` namespace.  In the rules, click Specify a hostname, and use `demo.mydomain.com`, of course with your own domain name.  Our target backend will be a workload, we'll choose our `rancher-demo` in the dropdown, and port `8080`. You could also remove the workload line, replace it by a service, and select your port in the dropdown.  Let's hit Save.

Within a few seconds, your ingress will be Active, click on the `demo.mydomain.com` hotlink.  You'll get an SSL certificate warning, as only the `rancher` subdomain has a certificate for now.  Don't worry, we'll take care of that next.  Just make sure all three pods are responding to requests, and enjoy the awesome load balancing built into Kubernetes.

Congrats, you've deployed containers on K8s.  It really is that simple!

## 10.5. Certificates
### 10.5.1. Staging
We definitely want our entire cluster behind SSL, and the best way to accomplish that is to get a wildcard certificate from Let's Encrypt.  First of all, we'll be good internet citizens and test with the staging environment, as we'll be using a DNS-01 challenge as opposed to HTTP-01 like when we got a cert for our `rancher` subdomain.  Again, we assume you're using Cloudflare here, you'll need to look up how to adapt to your DNS provider.

Within Cloudflare, create a custom API token, and give it All zones - Zone:Read, DNS:Edit permissions.  Write down the token, it won't be shown again.

Back in Rancher, we'll create a secret (within the *System* project, get Resources, than Secrets, and Add Secret top-right).  We're in *System* because that's where `cert-manager` lives.  Call it `cloudflare-api-token-secret`, make it available to the `cert-manager` namespace, and name the Key `apikey` and the Value will obviously contain the key.  Save it.

Now, we'll need to write some YAML.  Use [this template](/./cert-manager/ssl-staging.yaml) and adapt it with your info.  That's a ClusterIssuer, that'll take care of issuing every certificate you request.  On that, let's use [this next template](/./cert-manager/test-certificate.yaml) to create our test certificate.  Make sure to create a directory to save those two files, as we'll then use a command to deploy both to your cluster.

```
kubectl apply -f ssl-staging.yaml
```
Then
```
kubectl apply -f test-certificate.yaml
```

On Rancher, at the very top right, click the big *Try Dashboard* button.  If you're very curious, within your default Overview page, the bottom section, Events, should show - once you've filtered by Date - the associated events.  If you're not that curious, go pour yourself more of that beverage and wait 3-5 minutes, then under your Default project within Resources, then Secrets, head to *Certificates* on the sub-navbar and open your test certificate.  You should see a valid beginning date today, an expiry date 3 months from now, and both `mydomain.com` and `*.mydomain.com` in the Domain Names box.  Cool!

### 10.5.2. Production
Now we just need to do the same with actual SSL certs.  Refer to [this issuer](/./cert-manager/ssl-acme.yaml) and [this certificate](/./cert-manager/certificate.yaml) to obtain the real deal.

There is one issue - the certificates will only be issued to the *default* namespace.  I have an open issue with cert-manager to get more details, but at the time of writing this there is a workaround that will be kept up-to-date [here](/./useful%20commands/certificate-share.md), that we'll adapt for our case here.

```
kubectl get secret mydomain-com-tls -n default --export -o yaml | \
kubectl apply -n demo -f -
```

Let's go back to our `rancher-demo` ingress.  On the right hand side, click the three dot menu, then Edit.  Expand the *SSL/TLS Certificates* section, and select *Choose a certificate*, then get your wildcard cert in the dropdown, and reuse your `demo.mydomain.com` hostname under Host.  Save it, give it 5 seconds, and go back.  You should now be able to access your demo application and confirm it has a full SSL certificate.

## 10.6. Persistent Storage
Something awesome about Kubernetes is that you can seamlessly use storage across all nodes.  To set that up, we'll be using Rancher Longhorn.  It'll create a data pool by selecting as much free space as it can across all hosts, then replicate it across the hosts.  For example, if you have a mix of 500GB and 1TB free across your hosts, it'll create a ~500GB pool and make sure that pool is identical on each host, without you needing to do anything.  It's a beautiful way to address storage, makes sure your pods can move instantly from one node to another without data loss, and make backups a breeze.

Longhorn is an extremely powerful enterprise-grade product and we're not even scratching the surface today, at the time of writing this I'm still very much learning, so I encourage you to dig deeper.

From the navbar, go back the the main page by clicking Global in your cluster menu.  Then, select Apps, then Launch.  We'll search for/select Longhorn.  Leave all options as-is and click Launch.  It'll take ~5 minutes to set itself up. Once that's done, go back to your *local* cluster and *Default* project, then click Apps.  You'll see Longhorn, and there'll be a `/index.html` link.  Open it.

What you'll see is 1 volume, 3 nodes, and a total Storage Schedulable capacity.  You now have a persistent storage cluster across your K8s cluster.

## 10.7. What's Next?
We now have installed K3s on three nodes, creating a Kubernetes cluster.  We've setup Rancher on it, Longhorn for persistent storage, we've obtained a wildcard SSL cert for our ingresses, and deployed a workload.  If you feel comfortable with your cluster, feel free to stop reading here.  What we'll do next is:
- Secure our applications with SSO and 2FA using Authelia;
- Setup an easy way to reverse proxy to any service outside of your cluster in ~2 minutes, while keeping it safe behind SSO;
- Deploy a few more workloads that are more pertinent than flashing cows.

# 11. Securing our cluster
## 11.1. Why?
Really?  Joke aside, we'll run a lot of stuff here that could have no security, or more commonly low security.  I'm a big fan of high security applications, and by using [Authelia](https://github.com/authelia/authelia), an open-source authentication service, we'll be able to secure everything behind SSO.  Even better, Authelia will work out of the box with TOTP, FIDO U2F and Duo Push notifications for awesome, secure 2FA.

Unfortunately, Authelia doesn't yet pass authentication forward, which means you might have to login to Authelia first, and then to your service.  It becomes your choice to either bypass Authelia and rely on your services' authentication, do the reverse, or do both.

## 11.2. How?
Authelia is very configurable and supports many backends.  You can have a local users file or use LDAP, use an internal database or MySQL/Postgres, and store session info locally or within Redis.  My personal config uses QNAP LDAP, MariaDB and Redis, but I've used it for a few weeks using everything locally with good results.

It can run by itself for TOTP and FIDO U2F, but needs a Duo API key to do mobile push notifications.  I recommend it as it takes only a second to setup and, in my case, it's super wife-friendly.  As she logs in, she gets a Duo pop-up on her cell, taps the green checkmark, and a second later she's in.  I use a Yubikey, but I don't think I'll ever convince her to use one...

## 11.3. Setting up Authelia
### 11.3.1. Architecture
You'll need to decide your architecture first.  I won't go over the details, so please take some time to [go over the documentation](https://www.authelia.com/docs/).  Some people don't like instruction manuals, but hey, you got this far in this guide, so it's pretty clear you can do it :)  The thing is, the documentation regarding setting up Authelia in Kubernetes is non-existent as I write these lines.  I might tackle it later, but for now, I'll try to explain the differences.

No matter what your final choice is, we'll need to write more YAML.

### 11.3.2. The Configuration File
Everything in Authelia depends on the [config.yml](https://github.com/authelia/authelia/blob/master/config.template.yml) file.  It'll need to be setup before we install anything, and is mostly self-explanatory.  However, if you'd rather not put the secrets in that file (it's honestly very, very unlikely someone manages to get access to that file with how Kubernetes works), the documentation page about secrets has a config that doesn't work at all in K8s.  I currently have an open pull request that addresses it, but it's still pending as I write this.  For now, put the secrets in the file; I'll get back with a write-up on how to use K8s secrets and put a link here.  If you watch this repo, you'll be notified when I make changes, I'll create a release tag for each significant add.

Be aware you **must** configure SMTP if you want to setup TOTP, FIDO U2F or Duo Push at first login.  It's very easy to get a free SMTP provider, I personally use Google GSuite but a simple search got me a dozen results that'll let you send several hundred free emails per month, which is way more than enough as Authelia only emails you on 2FA setup and password changes.

### 11.3.3. Installing Authelia in Rancher
There's a little change in workflow here.  In Docker, we'd typically SSH/SFTP into the server, create a directory (if you've read the Authelia docs, you'll know the mount point of `configuration.yml` is `/config`) and dump our file in there.  In Kubernetes, that's not how it works.  We'll create a *Config Map*, which is basically a config YAML file, and mount it to our workload.

Let's start by creating a namespace called `authelia`.  Either use the webui or `kubectl create namespace authelia`. In your *Default* project, head to Resources and Config in the navbar.  Click Add Config Map, and name it `authelia-config`.  Be sure to select the authelia namespace.  Call the key `configuration.yml` and literally paste your entire file in the Value box.  Save it.  Done.

Go back to your workloads, and click Deploy.  Start with one pod, and use `authelia/authelia` as the image.  Don't forget the namespace.  Map port 9091 as a TCP Cluster IP, and add the TZ environment variable with the appropriate time zone notation as a value.  We'll need to create a Volume for Authelia to grab our configuration file.  Click the dropdown arrow on Add Volume, and select *Use a config map*.  Call it `config-volume`, choose your config map in the drop down, and put `/config` as the mount point.  See the logic?  Super simple.  Save and deploy.

Next step, you guessed it, is to create an ingress.  Start by duplicating your certificate into the authelia namespace as [per step 10.5.2](#1052-production).  Then, in Load Balancing and Add Ingress, reproduce what we did with the demo under the same 10.5.2 step.  You now have your login point for our whole cluster's SSO.  Isn't that awesome?

### Putting services behind Authelia
It's great to have SSO, but you have to make sure your deployments are actually protected by it.  Right now, they're not, but let's put our demo app behind it.  We'll use a simple set of annotations in the ingress to make sure the backend (which is actually Traefik 1.7) knows to send the request to Authelia first.  When you log in, you won't have to re-authenticate for the duration of your session in the configuration file.  It's therefore important to use an incognito window when you test, and close it after, to delete that session cookie.

#### Traefik Configuration
We'll need to add a few lines in the Traefik TOML config file.  Go to your *System* project, and access the config maps under Resources - Config.  Edit the `traefik` line that has a `traefik.toml` key.  You can edit it directly in the box, or copy-paste.  Here is what needs to be added:
```toml
    [entryPoints.http.auth.forward]
    address = "https://authelia:9091/api/verify?rd=https://authelia.mydomain.com"
    trustForwardHeader = true
    authResponseHeaders = ["Remote-User", "Remote-Groups"]
```
To see how it fits in the actual file, refer to [my example config](/./traefik/traefik.toml).

#### Ingress Configuration
Go back to your demo ingress, click Edit in the menu, and expand *Labels & Annotations*.  We'll add the following 5 annotations (**not labels!**), assuming your Authelia domain is `https://authelia.mydomain.com` :
```
kubernetes.io/ingress.class                         =   traefik
ingress.kubernetes.io/auth-type                     =   forward
ingress.kubernetes.io/auth-url                      =   http://authelia.mydomain.com/api/verify?rd=https://authelia.mydomain.com/
traefik.ingress.kubernetes.io/redirect-entry-point  =   https
traefik.ingress.kubernetes.io/redirect-permanent    =   true
```
A cool Rancher trick: if you copy this as-is to your clipboard, you can click Add Annotation once, and paste it in the first Key box.  Thank me later.

Now, test your demo service again by opening a private browser window and going to `http://demo.mydomain.com`.  You'll notice you're redirected to HTTPS, and you'll hit Authelia.  After you login, you'll be seemlessly redirected to the demo app.  How cool is that?

# Reverse Proxying
A very cool trick with our setup is that we can very easily reverse proxy to services outside of our cluster, while leveraging our SSL cert and Authelia SSO.  Even better, as a reverse proxy, you can go straight to a *specific port* from your subdomain!  Let's say you have a NAS that you really don't want to expose to the internet because you don't trust its login page.  You'll be used to access your NAS via IP, let's assume `https://10.0.1.10:8080`.  How nice would it be to use `https://nas.mydomain.com` from anywhere in the world without VPN and be protected via secure 2FA, with the bonus of having an actual SSL cert as opposed to your NAS's self-signed one?

## Create the Service
Within your Default project, click *Service Discovery* in the sub-navbar, then Add Record.  We'll name it `nas-redirect` and create a new namespace, `outside-redirects`, where we'll put all the records and ingresses to the outside.  Keep the Resolves To, and put your NAS IP, `10.0.1.10` **without the port** in the Target IP box.  Then, click the small *Show advanced options* link.

You'll want to select *Headless Service* in the dropdown, and then Add Port and publish the appropriate one, `8080` in our example.

Leave everything else as-is and save.

## Create the Ingress
We'll create a new ingress named `nas-redirect` in our previous `outside-redirects` namespace and specify our `https://nas.mydomain.com` hostname.

In the Rules, we'll need to remove the Workload already there with the `-` button and add a Service instead.  Choose our `nas-redirect` in the dropdown, and our `8080` port in the other dropdown.

The rest is the same, you'll select your SSL cert, and put the annotations in.

Test it, you should have a functional redirect!

# Conclusion
Subsequent versions of this tutorial will have some examples of more deployments: download services, media servers, databases, monitoring, dashboards, and so on.  Stay tuned!  For now, that concludes the guide and should give you a pretty good big-picture understanding of Rancher and how to deploy stuff on it.

## How can I help?
If you've enjoyed this guide, I'm really happy!  Please feel free to fork, open pull requests and/or issues, and adapt it to what you want to do.  If I messed up somewhere, please tell me!

Please enable notifications so you get notified when I update the guide, as there's more coming for sure.

If you've really, really enjoyed it and would like to buy me a coffee, it would be very nice of you!

<style>.bmc-button img{height: 34px !important;width: 35px !important;margin-bottom: 1px !important;box-shadow: none !important;border: none !important;vertical-align: middle !important;}.bmc-button{padding: 7px 15px 7px 10px !important;line-height: 35px !important;height:51px !important;text-decoration: none !important;display:inline-flex !important;color:#ffffff !important;background-color:#5F7FFF !important;border-radius: 8px !important;border: 1px solid transparent !important;font-size: 24px !important;letter-spacing:0.6px !important;box-shadow: 0px 1px 2px rgba(190, 190, 190, 0.5) !important;-webkit-box-shadow: 0px 1px 2px 2px rgba(190, 190, 190, 0.5) !important;margin: 0 auto !important;font-family:'Cookie', cursive !important;-webkit-box-sizing: border-box !important;box-sizing: border-box !important;}.bmc-button:hover, .bmc-button:active, .bmc-button:focus {-webkit-box-shadow: 0px 1px 2px 2px rgba(190, 190, 190, 0.5) !important;text-decoration: none !important;box-shadow: 0px 1px 2px 2px rgba(190, 190, 190, 0.5) !important;opacity: 0.85 !important;color:#ffffff !important;}</style><link href="https://fonts.googleapis.com/css?family=Cookie" rel="stylesheet"><a class="bmc-button" target="_blank" href="https://www.buymeacoffee.com/thehedgefrog"><img src="https://cdn.buymeacoffee.com/buttons/bmc-new-btn-logo.svg" alt="Buy me a coffee"><span style="margin-left:5px;font-size:24px !important;">Buy me a coffee</span></a><br></br>


Thank you so much for taking the time to read this to the end, it's been a blast writing it and I'll do more in the future for sure!

thehedgefrog