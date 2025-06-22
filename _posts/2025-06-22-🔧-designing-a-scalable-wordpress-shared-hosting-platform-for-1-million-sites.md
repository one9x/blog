---
layout: post
title: 🔧 Designing a Scalable WordPress Shared Hosting Platform for 1 Million Sites
date: 2025-06-22T07:38:13.487Z
comments: true
author: Raman
categories:
  - Architecture
  - Serverless
  - Wordpress
---
## 🧭 Problem Statement

Shared hosting is the default for most WordPress websites — but traditional approaches don’t scale well. We need a system that can:

* Support **millions of WordPress sites**.
* Keep **resource usage low**.
* Make **updates and maintenance easy**.
* Be **highly available**, **secure**, and **cost-effective**.

- - -

## 🧱 Architecture at a Glance

### 🔹 Separation of Compute and Storage

* **Compute Layer:** Stateless pods (NGINX + PHP-FPM) serve content.
* **Storage Layer:** Shared **CephFS** filesystem stores all WordPress content.
* **Database Layer:** Multi-tenant **Neon PostgreSQL** serverless databases for each site, with **PgBouncer** for connection pooling.

### 🔹 Read-Only vs Read-Write Access

* **Read-Only Pods:** Serve user traffic (frontend, visitors)  
* **Read-Write Pods:** Handle admin actions (uploads, plugin installs)

### 🔹 Symlink-Based Core Versioning

* WordPress core is shared across all sites using **symlinks**  
* Changing the core version is as easy as updating a symlink

- - -

## 📁 CephFS: Scalable, Shared Storage

We use **CephFS**, a POSIX-compliant distributed filesystem, to store:

* WordPress core versions (e.g. `/shared/wordpress-core/v6.2/`)  
* Each site’s unique content (`wp-content/`, media, themes, plugins)  
* All sites under a unified namespace (e.g. `/mnt/cephfs/sites/`)

**Benefits:**

* Shared access across many Kubernetes nodes  
* High availability, redundancy, and fault tolerance  
* Easy to manage millions of directories (one per site)

- - -

## 🔄 Symlink-Based WordPress Core

Instead of copying WordPress core for each site, we:

* Keep shared core versions centrally:
  		`
  		/shared/wordpress-core/v5.9/
  		/shared/wordpress-core/v6.2/
  	
  		
* Point each site’s `public_html/wp` folder as a **symlink** to one of the versions:
  		`bash
  		ln -sfn /shared/wordpress-core/v6.2 /mnt/cephfs/sites/foo.com/public_html/wp
  		
  	

**Why this rocks:**

* Upgrades are instant: just change the symlink
* Rollbacks are safe and fast
* Saves huge amounts of storage
* No impact to custom content or plugins

- - -

## 🗄️ Database Layer with Neon PostgreSQL

Managing millions of databases is no small feat. To handle this at scale, we use **Neon**, a serverless, multi-tenant PostgreSQL service built for massive workloads.

**How it works:**

* **Multi-Tenant by Design:** Neon allows to create separate databases (or schemas) for each WordPress site, providing strong data isolation and security. Each site’s domain is mapped to its dedicated Neon database automatically.
* **Connection Pooling with PgBouncer:** To prevent Neon from being flooded with thousands of short-lived connections, we run **PgBouncer**, a lightweight connection pooler. It reuses and multiplexes connections efficiently, keeping the database layer stable under high concurrency.
* **Auto-Provisioned `wp-config.php`:** During site creation, provisioning system automatically generates a standard `wp-config.php` file for each site, injecting the correct Neon database name, user, password, and host. This keeps things simple and fully compatible with WordPress and its plugins — no special hacks needed.
* **Dynamic Resolver:** Our Domain Resolver Service not only maps domains to storage paths but also returns the database connection details for the corresponding site, so PHP-FPM connects to the right Neon instance or schema on every request.

**Benefits:**

* Strong tenant isolation
* Minimal connection overhead
* Easy scaling with Neon’s serverless architecture
* Simple and consistent database management

- - -

## 🌐 Handling Domains Dynamically

With millions of domains, we can't maintain 1 million NGINX configs. So we created a **Domain Resolver Service**:

* Accepts a domain (e.g., `foo.com`)
* Returns the site’s root directory (e.g., `/mnt/cephfs/sites/foo.com/public_html`)
* Also returns the Neon database connection details
* Used by NGINX and PHP-FPM to serve the right content and data dynamically

- - -

### 📦 NGINX + PHP-FPM in Action

1. **NGINX receives request** for `foo.com`
2. Calls the **Domain Resolver API**
3. Sets `$document_root` based on the resolved path
4. Forwards PHP requests to **PHP-FPM** with correct `SCRIPT_FILENAME`

- - -

## 🚀 Performance & Scaling

To handle scale and optimize performance:

* Use **FastCGI Cache** in NGINX for dynamic pages
* Integrate with a **CDN** (like Cloudflare) for static assets
* Deploy **stateless read-only pods** for scaling frontend serving
* Isolate **admin pods** to handle writes safely
* Offload connection handling to **PgBouncer** and Neon’s built-in pooling

- - -

## ✅ Key Benefits

| Feature                         | Benefit                                            |
| ------------------------------- | -------------------------------------------------- |
| **Kubernetes + CephFS**         | Horizontally scalable and resilient infrastructure |
| **Symlinked WP core**           | Easy upgrades, efficient storage                   |
| **Dynamic NGINX root**          | Supports millions of domains without reloads       |
| **Neon PostgreSQL + PgBouncer** | Secure, multi-tenant DB with connection pooling    |
| **Separation of roles**         | Security and performance (read vs write pods)      |
| **Multi-tenant friendly**       | Clean, isolated site environments                  |

- - -

## 🧪 Next Steps

Want to build this yourself? Start with:

1. Deploying **CephFS with Rook** on Kubernetes
2. Setting up **NGINX + PHP-FPM** with CephFS mounts
3. Creating a lightweight **Domain Resolver API**
4. Automating **symlink management** for WordPress core
5. Provisioning **Neon projects/databases** via API
6. Running **PgBouncer** for connection pooling
7. Implementing **FastCGI caching** and CDN optimization

- - -

## 💬 Final Thoughts

This architecture lets you serve **millions of WordPress sites** with strong isolation, efficient upgrades, and low overhead — all without reinventing the wheel.

If you're building large-scale hosting platforms or cloud-native WordPress infrastructure, this approach gives you the flexibility of containerization with the robustness of traditional hosting — at serious scale.

- - -

## 📚 Resources

* [CephFS Overview](https://docs.ceph.com/en/latest/cephfs/)
* [Kubernetes + Rook](https://rook.io/)
* [PHP-FPM Explained](https://www.php.net/manual/en/install.fpm.configuration.php)
* [OpenResty Lua for NGINX](https://openresty.org/en/)
* [Neon Serverless PostgreSQL](https://neon.com/)
* [PgBouncer](https://www.pgbouncer.org/)
* [WordPress in Kubernetes (DigitalOcean)](https://www.digitalocean.com/community/developer-center)

- - -

**💡 Need help implementing this?**
Drop a comment or reach out — I'd love to hear how you're scaling your hosting infrastructure.