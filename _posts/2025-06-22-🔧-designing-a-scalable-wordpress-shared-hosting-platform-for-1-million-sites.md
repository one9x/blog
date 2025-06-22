---
layout: post
title: 🔧 Designing a Scalable WordPress Shared Hosting Platform for 1 Million Sites
date: 2025-06-22T07:38:13.487Z
comments: true
author: Raman
categories:
  - System Design
tags:
  - serverless
  - wordpress
  - neon database
  - kuberenets
---
Hosting a wordpress site is one of the first things many developers do. It feels straightforward:  either choose a wordpress hosting provider — click deploy, or: Spin up a server — install PHP, Nginx, Postgres/Mysql — Run famous 5-minute wordpress setup.

It's simple, cheap, and works perfectly  —  for **one** site.
But what if you want to host **Thousands sites**? or a million? Just like the hosting providers do.

Well it's easy to assume that just "Add more servers" and it magically scales, like it done in a system design youtube video tutorials. Unfortunately it doesn't work that way  — Let's try to understand and design "The Wordpress Hosting Platform  for million sites".

This discussion is divided into 3 parts
- Understanding the problem statement -  What and why?
- Though process - how we reach to a particular solution and what are the other solutions
- The solution


# 🧭 Problem Statement

Shared hosting is the default for most WordPress websites — but naive approaches don’t scale well.

What is **Naive approach** ?
- One VM hosts 10 sites.
- Each site has its own folder and its own database.
- You configure NGINX with 10 server blocks.
- Everyone is happy.

This **shared hosting** model — cheap, simple, and great for a few sites.

But growth is sneaky. A client’s blog goes viral, or a new customer signs up every hour. Suddenly:
- Traffic spikes crash the whole server.
- One misbehaving plugin chews up CPU, making every other site crawl.
- A client wants PHP 8.1, but another’s plugin needs PHP 7.4.
- Storage balloons because everyone uploads cat photos and high-res videos.
- You now have 200 server configs to maintain.

Let’s break down the hidden pitfalls:

-  **🐢 Performance tanks fast**
One busy site slows down every other site on the same box. You can buy more servers — but that spreads the problem, it doesn’t solve it.

- **🔒 Security and isolation are weak**
Shared servers mean one hacked site can expose dozens more. Cheap shared hosting rarely has real sandboxing.

- **🔄 Updates turn into nightmares**
Running a security update on WordPress core, or a popular plugin, for 5 sites is fine. Doing it for 10,000 sites? Now you’re awake at 2am writing bash scripts and praying nothing breaks.

- **💸 Costs spiral out of control**
Hosting businesses make money by squeezing as many sites as possible onto each machine — *without* each site needing dedicated CPU and RAM all the time.

- **⚙️ Scaling is manual and fragile**
Need to move a site to a less loaded server? Better update DNS, the NGINX configs, the database settings, and hope you don’t break SSL in the process.

### 🎯 **What We Really Want**

So what does the *ideal* solution look like?

- Every site runs fast, even if another site misbehaves.
- Storage is endless and reliable — with backups and redundancy.
- Updates happen once, safely, for everyone.
- Adding a new site takes seconds, not hours.
- Traffic routing works automatically — no hand-crafted server configs.
- You can scale from 1 site to 1 million sites without rewriting everything.
- Keep the costs in control and maximize the profits.

Sounds impossible? Not quite. Big hosting providers like WP Engine, Kinsta, or large SaaS platforms have cracked this at scale — by combining clever design, modern tools, and lessons learned from obervations.


# 🧭 The Thought Process

Before designing something that can host millions of WordPress sites, it’s worth stepping back to understand how a single site works — and how the naive approach evolves (and eventually breaks) as we scale.

Let’s start simple:

###  Setting Up a Single WordPress Site
- Provision a VM with the necessary stack: PHP, NGINX, and Postgres.
- Create a database for WordPress.
- Install and configure WordPress with the database credentials.
- Set up an NGINX server block for the site’s domain.
- (Optional) Add a public load balancer to route external traffic to the VM.

> For detailed step-by-step instructions, check out [How to Install WordPress](https://developer.wordpress.org/advanced-administration/before-install/howto-install/).

###  Expanding to Multiple Sites

Now, suppose you want to host a few more sites on the same server. The steps are conceptually the same:

- Create a new folder, install WordPress again
	```
	/var/www/site1.com/
	/var/www/site2.com/
	/var/www/site3.com/
	```
- Create an additional database (often on the same Postgres server).
-  Configure the new site to connect to the database.
- Add a new NGINX configuration for the new domain.

> Note: WordPress itself supports multiple instances on the same machine — see [Multiple Instance Installation](https://developer.wordpress.org/advanced-administration/before-install/multiple-instances/).*

This works nicely for a **handful of sites** — you can script or automate it with Infrastructure-as-Code (IaC) tools, keeping maintenance overhead low.

**✅ Pros of this naive approach**
- Easy to set up.
- Simple resource sharing.
- Low upfront cost.
- Straightforward to automate for a small fleet.

**❌ Where it starts to break?**
 as your hosting business grows, cracks appear:

- **Noisy Neighbor Problem:** One busy site can hog CPU, RAM, or disk I/O, degrading performance for others.
- **Scaling Bottlenecks:** A single VM or bare-metal server can only handle so much traffic and storage.
- **Maintenance Complexity:** If a server goes down for maintenance, *all* sites on it go down with it.
- **Traffic Routing Headaches:** Load balancers need to know which backend server hosts which site. Managing thousands of custom rules or an internal DNS server becomes cumbersome.
- **Resource Waste:** VMs often force you to allocate more CPU/RAM than each site actually needs, leading to inefficiencies.

### Adding More Servers

A common next step is to run multiple servers, each hosting many sites. This distribute the risk and increases capacity — but introduces new challenges:
- You must track which server hosts which sites.
- Routing must be kept in sync with your DNS or load balancer.
- If you want to migrate sites between servers, you need to handle downtime, IP changes, and storage migration.
- To smooth this out, some hosts use Virtual IPs (VIPs) or reassign public IPs to new servers — but this gets operationally messy at scale.


###  Enter Containers and Orchestration

Modern best practice is to move from VMs to containers:

- Each WordPress site runs in its own lightweight container, isolated and resource-limited.
- Kubernetes schedules and scales containers across your cluster.
- Ingress controllers or Gateway APIs handle traffic routing automatically.
- Provisioning, configuration, and upgrades can be managed declaratively (e.g., Helm charts + ArgoCD).

This brings better automation and security isolation — but with trade-off: 
- Persistent volume provisioning. As all of the cloud providers provides a abstraction over this we may think of this as free service without overhead, but to have this as hosting provider we have to implement a storage system like NFS backed by ZFS pool or CephFs which are additional sytems to manage. With number of solution avaliable for this it will not be a major techincal trade-off but significantly increase infra cost or working capital.
- Container-based multi-tenancy still consumes reserved CPU/RAM for each running site, whether or not it gets traffic.
	
> For more details on how kubernetes scheduler schedule a pod [read here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/). In short kubernetes scheduler looks for a node which has enough resources available based on resource limits defined for the pod, even if the active usage of the node is not that much if existing pods in the node has requested for all the resources available or available capacity is less than request, no new pod can be scheduled on that node

The hosting economics reality hosts stay profitable by **over-provisioning**, For example, a server with 10 CPU cores and 10 GB of RAM *should* fit 10 WordPress sites that each need 1 CPU and 1 GB RAM — but since not every site is busy all the time, you might actually fit 12 or more without issues.

If you use containers with strict resource requests and limits, you lose some of this oversubscription advantage — hurting your efficiency and profit margin. *No matter how good a solution techincally is, business always runs on profilts.*

###  The Big Waste: Duplicate WordPress Core

If we observe the containers and even classic server setups, we hit another waste:  
Every site had its *own copy* of the WordPress core files — `wp-admin`, `wp-includes`, etc.

Storage is cheap today — disks cost pennies per GB — but when you have millions of sites, those identical folders multiply fast:
- 50 MB per core copy × 1 million sites = ~50 TB of redundant files.
- More clutter to manage.
- Updating core for millions of folders means millions of file operations.

We needed a better way: one core version, shared for all sites, safely upgradable with a simple switch.

##  Can We Do Better?
Short answer : Yes

If we observe one of the best and efficient solution is a naive solution a single server with multiple sites hosted on it. What if we could make a compute and storage **practically limitless**?
- Storage big enough to cover all the storage requriements and fast enough to handle latency.
- Compute powerful enough to handle all the compute needs of overall system, which can scale up and down based on the load.

Let's try to solve for these:

### Storage
Today a single disk can have a capacity upto 20+ TB, which is not enough to meet the system requirement but on cloud we have 100 of TB's storage solutions available how does they do it?
Answer is distributed storages, there are multiple solutions already available for this like ZFS, CephFs, FlusterFs etc.
> A [distributed storage system](https://www.geeksforgeeks.org/computer-networks/distributed-storage-systems/) is a computing infrastructure designed to store and manage data across multiple interconnected nodes or servers. Unlike traditional centralized storage systems, where data is stored in a single location, distributed storage systems distribute data across a network of nodes

So now as we have a **practically limitless** storage system, all the sites can be installed on the it. With this we can easy solve "Duplicate WordPress Core" and "Upgrading wordpress core" problems. 
We version the wordpress core :
```
/shared/wordpress-core/v5/
/shared/wordpress-core/v6/
```

Each site’s `public_html/wp` folder is just a symlink:
```
ln -sfn /shared/wordpress-core/v6 /mnt/shared/sites/foo.com/public_html/wp
```

Upgrading everyone? Change the symlink — instant update. Rollback? Switch back.  
No more juggling millions of redundant folders.

Although as we now there is nothing called free launch, everything comes at a cost. If a single unified storage, the security of a single server getting compromised leads to whole system getting compromised still exits. In order to reduce the risk we can use another observation: 
Visitor traffic is mostly read-only (for file system, write action are to the database), but admin actions (uploads, plugin installs) need writes.

So we split file system access pattern by role:
- **Read-Only:** serve visitors, cache pages, mount storage read-only.
- **Read-Write:** serve admin dashboard, uploads, and site changes with write access.

This split:
- Protects content.
- Isolates heavier backend actions.
- Makes frontend autoscaling safer and cheaper.

### Compute
In general if we observe the services they are of  two type: Stateful and Stateless.
- Stateful services: are those service which need a maintain a local state to process request.
- Stateless service: are those service which doesn't depend on any local state to process the request.

So stateful services needs all the requests to be processed by a single server which has the respective state, while stateless service don't have such constrain and any server can process any request.

If we observe how wordpress or php works, it's a stateless service. That means we can send any request to any server provided that server has access to the request resource. If every server mounts the same shared file system, then *any* server can handle requests for *any* site. 

This means:

- Idle sites consume zero compute.
- Busy sites automatically use available compute.
- No server is tied to specific sites — everything is fluid and dynamic.
- Dynamically scaling compute servers based on overall system usages instead of particular site usages, managed via kubernetes

# The solution
If we stitched everything together - The practical blueprint of a wordpress hosting system built to handle millions of sites is ready.

## 🧱 Architecture at a Glance

- **Compute is stateless** — any pod can serve any site.  
- **Storage is shared, redundant, and infinite-ish.**
- **Read only servers:** Serve user traffic (frontend, visitors)  
- **Read write servers:** Handle admin actions (uploads, plugin installs)
- **WordPress core is versioned once, symlinked everywhere.**  
- **Databases are multi-tenant and connection-pooled.**

## ⚙️ **How It’s Built**

🏗️ **Kubernetes for Compute**
- Pods run **NGINX + PHP-FPM** statelessly.  
- Ingress or Gateway API handles routing.  
- Pods auto-scale with load.

🔑 **Read vs Write Pods**

To keep performance high and secure:
- **Read-Only Pods** serve cached pages, mount storage read-only, and scale freely.
- **Read-Write Pods** serve admin actions (uploads, plugin installs), mount storage with write permissions.

This handles huge visitor traffic while keeping backend actions secure and predictable.

🗂️ **Shared Storage**

Options:
- **CephFS:** Highly scalable, self-healing.
- **NFS + ZFS:** Simpler for smaller clusters.
- **Other distributed filesystems:** GlusterFS, Longhorn, cloud-native stores.

Each site’s folder contains unique content; the WordPress core is symlinked.  No more duplicating 50 MB per site for millions of sites.

🗄️ **Database**

Managing millions of databases is no small feat. To handle this at scale, we can use **[Neon](https://github.com/neondatabase/neon)**, a serverless, multi-tenant PostgreSQL service built for massive workloads.

**How it works:**
* **Serverless** Similar to our wordpress hosting system neon is a serverless postgres, which has seperate storage and compute. This helps with managing resources better with only active sites consuming compute resources.
* **Multi-Tenant by Design:** Neon allows to create separate databases (or schemas) for each WordPress site, providing strong data isolation and security. Each site’s domain is mapped to its dedicated Neon database automatically.
* **Connection Pooling with PgBouncer:** To prevent Neon from being flooded with thousands of short-lived connections, we run **PgBouncer**, a lightweight connection pooler. It reuses and multiplexes connections efficiently, keeping the database layer stable under high concurrency.

**Benefits:**
* Strong tenant isolation
* Minimal connection overhead
* Easy scaling with Neon’s serverless architecture
* Simple and consistent database management


**🌐 Handling Domains Dynamically**
With millions of domains, we can't maintain 1 million NGINX configs or ingress rules. 
So we created a **Domain Resolver Service**:
* Accepts a domain (e.g., `foo.com`)
* Returns the site’s root directory (e.g., `/mnt/cephfs/sites/foo.com/public_html`)
* Used by NGINX and PHP-FPM to serve the right content and data dynamically

**How it works:**
1. **NGINX receives request** for `foo.com`
2. Calls the **Domain Resolver API**
3. Sets `$document_root` based on the resolved path
4. Forwards PHP requests to **PHP-FPM** with correct `SCRIPT_FILENAME`


## ⚙️ **Provisioning a New Site**

In a nutshell:
1. Create folder on shared storage.
2. Symlink the desired core version.
3. Create DB in Neon.
4. Generate fresh `wp-config.php`.
5. Register in Domain Resolver.
6. DNS points to your global load balancer.

## ✅ **What Works Well**

| 🔑 What We Built | ✅ Why It Works |
|------------------|------------------|
| **Stateless pods** | Easy to scale and upgrade |
| **Shared storage** | One source of truth, resilient |
| **Symlinked core** | Tiny storage footprint, instant updates |
| **Dynamic routing** | Add sites on the fly, no reloads |
| **Multi-tenant DB + pooling** | Efficient connections at scale |
| **CDN + FastCGI Cache** | Fast delivery for visitors |

## 🎯 **The Trade-Off**

No system is flawless — here are the *real* trade-offs unique to this approach:

### 🗂️ **1️⃣ Shared Storage Needs Careful Tuning**

Using a single distributed file system for millions of sites means:
- Heavy traffic spikes or huge file uploads can stress IOPS and metadata performance.
- Requires planning: right replication, caching layers, and capacity monitoring.
- Misconfigured quotas can let a single busy site eat up storage bandwidth.

The benefit: it works — but shared storage performance must be tested and scaled properly.

---

### 🌐 **2️⃣ Dynamic Routing Adds a Moving Part**

Resolving millions of domains on the fly means:
- You need a robust, fast **Domain Resolver Service**.
- If that resolver fails or gets overloaded, routing breaks.
- Must design for caching and graceful fallback.

The benefit: zero static configs and easy automation — but you’re trading simplicity for flexibility.


### ⚙️ **3️⃣ Ops & Automation Become Critical**

This design heavily depends on:
- Automated site provisioning (folders, symlinks, DB creation).
- Automated core version management.
- Good CI/CD for storage, resolver, and DB config changes.

If your automation breaks, manual fixes can be tedious.

In short:  
**This system trades some hardware cost and complexity for massive scale, easier upgrades, better site isolation, and automation.**

If you run a serious hosting business, that trade usually pays off fast — but it’s not magic. You still need good ops practices, alerts, backups, and smart scaling rules.

## 🚀 **What’s Next**

Want to build this?
- Start small: prototype with a simple NFS  on a dev cluster.
- Test symlinked core updates.
- Experiment with Neon and PgBouncer for multi-tenancy or use postgres database and create multiple database for each tenant.
- Design a robust Domain Resolver — keep it simple at first.

Got ideas or want to share how *you’d* tweak this?  
**Drop a comment — I’d love to hear your take! 🚀**

## 📚 Resources

* [CephFS Documentation](https://docs.ceph.com/en/latest/cephfs/) — scalable POSIX-compliant distributed filesystem.
* [Kubernetes + Rook](https://rook.io/) — run Ceph and other storage backends on Kubernetes.
* [NFS & ZFS Basics](https://openzfs.org/) — robust file storage alternative for smaller clusters.
* [PHP-FPM Configuration](https://www.php.net/manual/en/install.fpm.configuration.php) — how PHP-FPM works and how to tune it.
* [OpenResty Lua for NGINX](https://openresty.org/en/) — dynamic routing logic with Lua inside NGINX.
* [Neon: Serverless Postgres](https://neon.com/) — cloud-native multi-tenant Postgres with branching.
* [PgBouncer Connection Pooler](https://www.pgbouncer.org/) — lightweight Postgres connection pooling.
* [WordPress Kubernetes Guide (DigitalOcean)](https://www.digitalocean.com/community/tutorials/tag/wordpress) — examples and best practices.
* [Helm Charts for WordPress](https://artifacthub.io/packages/helm/bitnami/wordpress) — deploy WordPress on Kubernetes using Helm.


**✨ Thanks for reading — happy scaling, and may your sites never crash!**
