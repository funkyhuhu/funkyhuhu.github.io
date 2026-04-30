---
title: "Streamlining Docker Compose YAML: Using Anchors and Aliases"
date: 2026-04-30T16:00:00+08:00
draft: false
description: "This guide details how to leverage YAML Anchors and Aliases to optimize your Docker Compose configuration files."
tags: ["Programming", "Deployment", "Docker"]
series: ["Deployment"]
---

When maintaining complex Docker Compose projects, it is common to encounter scenarios where multiple services share the same image, networks, or environment variables. Manually writing out the configuration for every service not only leads to bloated files but also increases the risk of missing an update when changes are required.

To eliminate this redundant effort, the most efficient approach is to use native YAML features: Anchors (&) and Aliases (*).

# 1. The Pain Point: Redundant Configuration
When multiple services (such as a Web app and a Worker) share a core configuration, a typical YAML file looks like this:

```YAML
services:
    web:
        image: my-app
        environment:
            DEBUG: "true"
            DB_HOST: "postgres"

    worker:
        image: my-app
        environment:
            DEBUG: "true"
            DB_HOST: "postgres"
```
The downside is clear: if the DB_HOST needs to change, you must manually update it in two different places simultaneously.

# 2. The Solution: Define Templates and Reference Them
By using YAML Anchors, we can define a core template that other services can inherit from and override as needed.

## Optimized Implementation:
```YAML
# 1. Define the template (Starting with x- is a Docker Compose convention for custom extensions that aren't parsed as services)
x-common-config: &backend-common
    image: my-app
    environment:
        DEBUG: "true"
        DB_HOST: "postgres"

services:
    web:
        <<: *backend-common # 2. Reference and expand the template content
        environment:
            <<: *backend-common # Expand the environment variables
            PORT: 8000          # Add additional variables specific to the web service

    worker:
        <<: *backend-common # 3. Reference directly without extra configuration
```

# 3. Three Key Benefits of This Optimization
## Structured Management (Anchors & Aliases)
Use `&` to define an anchor name and `*` to reference it. The `<<:` symbol represents a "merge," which unpacks the template content into the current block while allowing you to add service-specific configurations below it. For large-scale projects, this typically reduces file length by over 40%.

## Network and Permission Minimization
While optimizing logic, it is best practice not to expose all services to the same network.

Internal Services: Services like worker or database initialization scripts should only be attached to an internal network.

External Services: Only services that need to provide HTTP access (e.g., `web` or `flower`) should include external networks like `traefik-public`.

The Approach: Set the default base network in x-backend-common, then manually add specific networks at the individual service level.

## Single Source of Truth
Now, if you need to add a global environment variable, you only need to modify it once in the x-backend-common block. All services inheriting from that template will update automatically. This significantly reduces maintenance overhead and the margin for error.

# 4. Conclusion
Using Anchors and Aliases is about more than just "typing less." It is about building a deployment architecture that is easy to maintain and logically sound. For Docker environments running multiple Celery workers or microservices, this is an indispensable technique.