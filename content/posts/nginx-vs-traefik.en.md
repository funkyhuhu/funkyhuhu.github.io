---
title: "Why I Swapped Nginx for Traefik: Deconstructing Cloud Automation and Development Environments"
date: 2026-05-01T04:00:00+08:00
draft: false
description: "This document details the differences between Nginx and Traefik, as well as cloud deployment automation and development environments."
tags: ["Programming", "Nginx", "Traefik"]
series: ["Programming"]
---

When handling GCP VM deployments, the most intuitive approach is usually installing Nginx as a Reverse Proxy and manually applying for Let’s Encrypt certificates. However, when a project requires frequent iterations and must balance "local development experience" with "cloud automation," this traditional method feels overly cumbersome.

This article shares how I built a self-sufficient infrastructure using GCP + Docker Compose + Traefik, along with my thought process regarding "environment consistency" and "network routing."

# 1. Tech Stack Selection: Why Traefik Over Nginx?
In my architectural design, the traffic path is: `Internet -> Traefik -> web (App)`.

A common question arises: Since the web container already has Nginx handling static files, why not just add another layer of Nginx on the outside?

## 1. Thoughts on Dynamic Configuration
When using traditional Nginx as a Reverse Proxy, every time I add a service or change a domain, I have to manually modify the `.conf` file and run a `reload`. In contrast, Traefik is designed specifically for the Docker ecosystem; it can automatically discover services by listening to Docker Labels.

## 2. Automated HTTPS Certificate Management
Traefik has built-in support for the ACME protocol (Let’s Encrypt). On a VM, I only need to define the `ACME_EMAIL`, and it handles certificate requests and renewals automatically. This allows me to focus my energy on feature development rather than maintaining server certificates.

# 2. Deployment Challenges: Elegantly Bridging "Dev" and "Prod" Environments
The biggest pain point between local development (Local) and cloud deployment (VM) is this: I want to test Traefik's routing logic locally, but I don't want to deal with HTTPS certificates on my machine.

**The Problem:** If I only write one `docker-compose.yml`, I would have to constantly modify Labels to switch between local domains and Production HTTPS settings. This is extremely prone to accidental deletions or configuration errors.

**The Solution:** Use the Override mechanism to achieve "Configuration Inheritance." I chose to leverage Docker Compose's default override behavior:
> - `docker-compose.yml (Base)`: Defines the Production environment. Uses the real domain and enables HTTPS.
>
> - `docker-compose.override.yml (Local)`: Executed only on the local machine.

- **Override Production Settings:** Disable HTTPS and switch to plain HTTP.
- **Reroute Traffic:** Use `localhost.tiangolo.com` (a ready-made domain pointing to `127.0.0.1`) to ensure a full domain testing environment during development.
- **Accelerate Development:** Directly mount the local `index.html` into the container. This enables instant updates without the need to repeatedly rebuild the image.

**Decision Logic:** On the VM, I force the command `-f docker-compose.yml` to ignore the override file; locally, I simply run `docker compose up`. This "whitelist" style of management ensures the stability of the live environment.

# 3. Practical Details of GCP VM Deployment
When deploying to a GCP VM, I don't manually log in to copy files. Instead, I manage deployment parameters through environment variables.

## Update Logic
For image updates, I adopt a `pull` -> `up -d` strategy. This ensures that Docker prioritizes fetching the latest tags from the Registry and performs a painless switch at the container level, minimizing service downtime.

# 4. Conclusion: Thinking Matters More Than Tools
Technical selection shouldn't just be because "a certain tool is popular," but rather based on what problem it solves.

Choosing Traefik was about lowering maintenance costs; choosing Compose Override was about maximizing development efficiency. Through this architecture, I achieved:

- **Infrastructure as Code (IaC):** All routing logic is written within Compose Labels.
- **High Environment Consistency:** The routing logic running locally is nearly identical to the cloud, differing only in encryption.
- This is where I spent the most time thinking during the early stages of the project: How to build a deployment environment that makes "future me" feel at ease?