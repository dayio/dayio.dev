+++
date = '2026-02-17T22:49:15+02:00'
draft = false
tags = ['docker', 'kubernetes', 'devops', 'swarm']
title = 'Why Docker Swarm is still My Go-To in 2026'
+++

I still remember the early days of containerization. Back then, I deployed a DC/OS cluster for production apps with Marathon, all running on top of Mesos. Kubernetes already existed, but it was incredibly complex for my needs and too hard to set up.

I followed the trends and the evolution of K8s for a long time. I wrote manifests for all my apps, set up CD pipelines with ArgoCD and FluxCD (why not both ?), and tested managed K8s with autoscaling. But something just felt off. I’m a "simple" backend engineer who does a lot of sysadmin and DevOps work, and Kubernetes simply felt like overkill.

And that’s when Docker Swarm clicked for me.

## The "PHP Treatment" of Docker Swarm

I had often read that "Docker Swarm is dead." But the truth is, people are just treating it like PHP. Hardly anyone codes in PHP 5 anymore; it’s a proper, modern language now with much lower overhead than many other interpreted languages, yet the myth persists.

The same goes for Swarm. Everyone thinks about the add-on named "Docker Swarm Mode" that is now deprecated. But, since Docker 1.12, Swarm is a built in feature and is maintained, updated and improved.

Even on a single node, you get all the deployment and recovery features of Swarm, along with the ability to later deploy an unreachable private instance to handle workload. The service mesh will automatically route the traffic without needing to open a port, which is quite nice. I genuinely hope 2026 will be the year we see stronger adoption and a spotlight put back on Swarm.

## Stateless by Design: Cold Starts are a Feature

On my side, I run multiple Laravel e-commerce applications with a lot of features (API, CRM, ERP, CMS), a massive Laravel WMS managing tens of thousands of stored products, a Node.js image proxy for on-the-fly resizing, and various data analysis tools.

Since I started coding, I have always treated my applications as stateless. I don't like dealing with stored state or virtual machines (JVM). I prefer accepting a cold start boot time penalty to ensure each request yields the same result, rather than spending time debugging stateful applications or storage issues. Cold start penalties are a feature, not a bug (and can be fixed with caching). Furthermore, I don't like the idea of vertical scaling too much because it often masks the flaws in the code (like memory leaks).

Not having built-in storage forces us to think carefully about how we store data, rather than blindly trusting StatefulSets until the moment we need to back up or restore. Another strict rule for me is to not manage the database myself. Doing HA, backups, and updates in production is always stressful. Outsourcing this to a managed service is the ultimate peace of mind.

## Disaster Recovery in 15 Minutes

When an issue occurs, the recovery process is highly acceptable. With sufficient backup and a tolerance of 10 to 15 minutes of downtime, disaster recovery is actually faster than a full Kubernetes DR.

You just spin up a new instance, install Docker, remap storage, deploy, update DNS (or flexible IP), and go. Personally, I use Scaleway for hosting. I leverage instance snapshots and cloud-init files. I just need to SSH, launch `docker swarm init`, and it's ready. Ansible is also incredibly good at handling provisioning and disaster recovery for this kind of setup.

## Building the Missing Management Tool

I’ve tested Dokploy, Dockge, Komodo, Portainer, Dockhand, and many others, but none of them felt exactly right.

So, I’m currently coding my own management tool. It handles stack deployment, scaling, aggregated logs, and rollback management, exclusively for Docker Swarm. It's remarkably easy, initially only needing `/var/run/docker.sock`.

I already built a working version with Laravel and Bun for the agent, but I was not satisfied with the way data flows across the stack and the lack of real-time capabilities. So I pivoted and started a new project with SvelteKit. The folks at Dockhand use it, and I found it quite pleasant to read. The data flows are logical and straightforward.

The architecture is simple: a Bun server handling SvelteKit, spawning a WS server. The agent will broadcast its presence, events, logs, and TTY on demand, and the server will push stacks or desired states. This architecture requires the server to be reachable externally, but the massive gain is that I don't have to open management ports on the cluster itself.

If you want to tinker with Swarm yourself, Docker-in-Docker is incredibly useful for local testing. Here are the compose files and scripts I use to spin up local clusters:

- [Single node cluster setup](https://gist.github.com/dayio/463b354d00f02b4c8bc585cd6dbb2374)
- [Multi node cluster setup](https://gist.github.com/dayio/5c1ac3e2807f7556f5623f50306d69b7)

Kubernetes has its place, but for the vast majority of us building and shipping products, Swarm is the pragmatic, stress-free choice.
