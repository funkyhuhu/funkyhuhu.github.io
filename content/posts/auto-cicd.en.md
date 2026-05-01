---
title: "Building a Rollback-Capable CI/CD with GitHub Actions + Docker: Practical Insights from Build to Production"
date: 2026-05-01T04:13:00+08:00
draft: false
description: "This document details how to implement a rollback-capable CI/CD pipeline using GitHub Actions and Docker."
tags: ["Programming", "Deployment", "docker", "CI/CD"]
series: ["Deployment"]
---

In software development, writing code is often not the hardest part; the real challenge is "ensuring that the code runs stably on the server."
This time, I didn't just "throw a website onto a server." Instead, I streamlined a complete deployment path into a repeatable process:

- Validate once a PR is opened.
- Automatically build the image on the `main` branch.
- Push to Docker Hub.
- GitHub Actions connects to the VM via SSH.
- Run `docker compose pull` and `docker compose up -d` on the server.

Connecting these tools might seem straightforward, but the real difficulty never lies in the tools themselves—it's about whether the separation of responsibilities between each layer is clearly defined.
Ultimately, my main concern wasn't just "did it deploy successfully," but rather: if this path fails, can I quickly identify exactly where it broke?

To save time in the early stages of a project, many developers (including my past self) fall into the habit of "manual deployment":
Manually running `build Docker image`, manually `pushing` to Docker Hub, `SSH-ing` into the server, and rebuilding the project.
As the project scales, this approach leads directly to three disasters: untraceable versions, environment inconsistency, and the near-impossibility of performing a fast rollback.

This article shares how I re-evaluated my CI/CD architecture to solve the core pain points of the deployment process.

> **Deployment should be a repeatable, traceable, and recoverable engineering process, not a manual operational one.**

# 1. The Decision: Why GitHub Actions + Docker?
Technical choices shouldn't be about chasing trends; they should solve specific problems with a clear division of authority.

**GitHub Actions**: Standardizing the Build—deciding when to build and when to deploy.
My reason for choosing GitHub Actions is simple:

> **Native Integration**: It is deeply tied to the Git Repository. When code is the "single source of truth," event-driven automation is the natural result.
> **Reproducibility**: CI should not happen on "my machine," because my machine has specific environment variables and caches. Through GitHub Actions, we force the build process to occur in a clean, reproducible virtual environment.

**Docker**: Eliminating "it works on my machine" and preserving fetchable artifacts.
Before adopting containerization, the most common issues were dependency version conflicts or runtime inconsistencies.
The core value of Docker to me is that it turns the execution environment into a portable "Artifact."

**Docker Compose**: The sweet spot for declarative deployment—responsible only for "running the specified image version."
Why not Kubernetes? For small to medium-sized production environments, the maintenance cost of K8s far outweighs the benefits.
I chose Docker Compose to achieve **Declarative** deployment:
Using YAML to describe the *state*, rather than using Shell scripts to describe *actions*.
Deployment is no longer about executing a sequence of commands; it’s about telling the server: "Please ensure the current environment state matches this configuration file."

By decoupling them this way, CI and CD do not contaminate each other. GitHub Actions doesn't need to know too many server details, and the server doesn't need to participate in the build.

# 2. Core Problems and Redesign
During the optimization process, I encountered three landmark problems that essentially pointed to the same architectural flaw.

## Problem 1: Unpredictability caused by the `:latest` tag
Early pipelines usually just `Push image:latest`. While convenient, this is suicidal for production.

- **Pain Point**: `latest` is constantly overwritten. You have no way of knowing which specific commit is currently running on the server.
- **Reflection**: A lack of **Immutable Versioning**. If the version changes under the same tag, it cannot be tracked, let alone rolled back.

**Solution**: Implement an **Immutable Image Strategy**.
Now, every version tag is unique (e.g., `sha-7ab2f1` or `v1.2.3`).
Each version is an immutable artifact. This solves the rollback problem—rolling back is simply "selecting an older tag," without needing to rebuild old code.

## Problem 2: Confused CI/CD Responsibilities
I previously tried to have CI handle both the build and the deployment logic (updating server settings).

- **Pain Point**: When the CI failed, I couldn't distinguish if the build was broken or if the server connection failed.
- **Reflection**: **Separation of Concerns**. The CI's mission should end at producing the image; it shouldn't interfere with the complex state of the runtime.

**Solution**: **Pull-based Deployment Workflow**.
I simplified the process:
**CI Phase**: Validate, package, and push the Image with a Hash Tag to Docker Hub.
**CD Phase**: The server retrieves the target tag via environment variable injection, then executes `docker compose pull` and restarts.
**Core Summary**: Deployment is no longer a process of "overwriting," but a process of "switching."

# 4. Final Architecture Flowchart

{{< mermaid >}}
flowchart TD
    A["GitHub Push / Tag"]
    B["GitHub Actions (CI)"]
    C["Build Docker Image"]
    D["Push to Docker Hub"]
    E["SSH to Server (CD)"]
    F["docker compose pull"]
    G["docker compose up -d"]
    H["Running New Version"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
{{< /mermaid >}}


# 5. Conclusion

The point of CI/CD isn't about which fancy tools you use; it's about whether you've established a predictable, reproducible, and traceable release process.

When you shift your deployment logic from "Imperative (manually doing X)" to "Declarative (I want state X)" and ensure version safety through Immutable Artifacts, you truly gain control over your system. Now, even if a bug is released, I can roll back to any historical version within 30 seconds. That sense of security is the true value of an automated architecture.

If you are also experiencing the pain of manual deployment, try starting with "Image Versioning"—it's a small change that makes a huge difference!