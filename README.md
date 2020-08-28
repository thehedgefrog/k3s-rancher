# Kubernetes Cluster for Home Use

## Why?
I've had a Docker-Compose stack at home for a while, running on an Ubuntu 18.04 VM on a QNAP NAS.  While the NAS can absolutely handle it with its 40GB of RAM, it creates a single point of failure.

I have 25+ containers running at any one time, and sometimes will scale up to 50+.  It's fair to say a lot of stuff in the home relies on this stuff.  Once in a while, I have to take down the NAS for maintenance (QNAP has software updates easily once a month) and that brings the entire stack down.  Let's just say it doesn't help with the Wife Approval Factor when the media server, the home automation and all the systems I've integrated in the house just stop working for 15-30 minutes.

I tried Docker Swarm, but I didn't like it.  It's limited, and quite frankly it's not the future, Kubernetes is.  I knew nothing about it, at all, when I started this project.  This project is documenting my journey learning it.

### <ins>DISCLAIMER</ins>
I'm pretty tech-oriented, and while I'm a K8s novice, I'm pretty good with Docker, networking, and computer technology in general.

This guide should be seen as a general reference, not a walkthrough.  It assumes you have a good understanding of Docker, Docker-Compose and running several Docker containers at once.  It also assumes you are well-versed in networking and understand the fundamental concepts.

## Objectives
- To learn K8s and their deployment.
- To run a stack of home services on Kubernetes via an HA K3s cluster.
- To learn and leverage Rancher for home use.
- To setup Rancher Longhorn for reliable persistent storage.

## Stretch goals
- Replicate the service on a cloud instance for more HA.
    - ETA: TBD