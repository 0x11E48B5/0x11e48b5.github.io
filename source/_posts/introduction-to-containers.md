---
title: What is container technology and how can we use it?
date: 2025-02-11 15:39:12
tags: ["devops", "docker", "container"]
---

# What are Containers?

Containers are something like a virtual machine, it has its operating system and all, except that unlike virtual machines `they don’t simulate the entire computer`, but rather create a sand boxed environment that pretends to be a virtual machine. Another defination for that; Containers provide a way of creating an isolated environment, sometimes called a sandbox, in which applications and their dependencies can live.

{% asset_img what-is-container-1.png %}

## Why do we need containers?

Containers are essential in IT because they provide a lightweight, portable, and efficient way to package, deploy, and run applications across different environments. They help developers and IT teams streamline software deployment, improve scalability, and enhance security.

### Key Reasons for Using Containers

1. Portability
Containers bundle an application and its dependencies into a single unit. This ensures that the application runs consistently `across different` environments (development, testing, production) without compatibility issues.

💡 Example: A developer can build an application on their laptop using Docker and deploy the same container to a cloud server `without modifications`.

2. Lightweight and Efficient
Unlike traditional virtual machines (VMs), containers `share the same OS kernel` but run in isolated user spaces. This makes them:
- Faster to start
- Less resource-intensive
- More efficient in scaling applications

💡 Example: A VM might require 2GB of RAM to run a simple web app, while a containerized version could use just 500MB.

3. Scalability and Microservices Architecture
Containers make it easier to `scale applications` by running multiple instances dynamically. They also support microservices architecture, where different parts of an application run in separate containers.

💡 Example: A web application might have separate containers for the frontend, backend API, and database, each scaling independently as needed.

4. Isolation and Security 
Each container runs in its own isolated environment, reducing the risk of conflicts between applications and improving security.

💡 Example: If one container is compromised, the attack is contained and does not affect the host system or other containers.

Why do we need this technology? In order to answer that question, we should know the difference between VMs and Containers.

## Difference between VMs and Containers

Both Virtual Machines (VMs) and Containers enable application deployment in isolated environments, but they differ in architecture, efficiency, and use cases.

| **Feature**        | **Virtual Machines (VMs)**  | **Containers**  |
|--------------------|----------------------------|----------------|
| **Isolation**     | Fully isolated, runs its own OS  | Process-level isolation, shares host OS kernel |
| **Size**         | Large (GBs), includes OS  | Lightweight (MBs), shares OS dependencies |
| **Startup Time**  | Slow (minutes) | Fast (seconds) |
| **Resource Usage** | High (each VM runs a full OS) | Low (shares OS, minimal overhead) |
| **Performance**   | Lower (due to full OS emulation) | Higher (less overhead) |
| **Portability**   | Less portable, tied to hypervisor | Highly portable, runs consistently across environments |
| **Security**      | Stronger isolation (separate OS) | Less isolated but can be hardened |
| **Use Case**     | Running multiple OSes, legacy applications, full VM environments | Microservices, cloud-native apps, fast deployments |


1. Architecture
VMs run on a hypervisor, which creates virtual hardware for each VM, requiring a separate OS instance.
Containers share the host OS kernel, isolating applications but using fewer resources.

💡 Example: Running 10 VMs requires 10 OS instances, while running 10 containers only requires one OS.

2. Performance & Efficiency
VMs have higher overhead due to the need for a full OS and virtualized hardware.
Containers are lightweight because they only package applications and necessary dependencies.

💡 Example: A VM may need 2GB RAM, while a container running the same app might need only 200MB.

3. Deployment & Scalability
VMs take longer to start since they boot an entire OS.
Containers start almost instantly, making them ideal for scaling applications dynamically.

💡 Example: A cloud service can deploy 100 container instances in seconds, whereas booting 100 VMs would take much longer.

Here is the picture that shows the difference between VM and Container technology

{% asset_img container-vs-vms.png %}

## What is namespaces and importance of it in containers?

Namespace is a feature in Linux that lets you see a specific part of the system, meaning allocating resources in an isolated environment. Namespace allows you to create that isolated environment where the container only knows what it can see because it's only in a certain namespace.

When you initialize a container, docker generates a set of namespaces for the container and every container has its own unique set of namespaces.

The purpose of each namespace is to wrap a particular global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the global resource. One of the overall goals of namespaces is to support the implementation of containers, a tool for lightweight virtualization (as well as other purposes) that provides a group of processes with the illusion that they are the only processes on the system” — https://lwn.net/Articles/531114/

Container engines use several namespaces, including:

- **PID** — Isolation of the pids allocation and list of processes, allowing containers to have their own process tree.
- **NET** — Isolation of network-related resources like network interfaces, iptables, and firewall rules.
- **IPC** — Isolation of the Inter-Process Communication resources.
- **MNT** — Managing mount points. Allows containers to have their own mount points and thus their own filesystems.
- **UTS** — Isolation of host and domain names. Allows containers to believe they’re running on differently named servers.

## What is Cgroups
Cgroups are another essential feature for containers. A cgroup is a Linux kernel feature (since v2.6.24) that limits, accounts for, and isolates resource usage (CPU, memory, disk I/O, etc.) for a group of processes. You can think of them as traffic cops for your programs that make sure no one takes or uses too much resources. Cgroups enable sharing the available hardware resources with containers and enforce usage limits.

Container engines use the following cgroups:

- Memory — Managing memory allocation and usage.
- HugeTBL — Accounting usage of huge pages by a process group.
- CPU — Managing CPU time and usage.
- CPUSet — Binding a group of processes to specific CPUs.
- BlkIO — Measuring and limiting the amount of I/O operations by group.
- net_cls and net_prio — Classifying and assigning network priorities to the traffic for traffic control.
- Devices — Managing read/write access devices.
- Freezer — Freezing a group. Useful for checkpointing (saving the state of a process) and migration.

With namespaces and cgroups, containers operate within well-defined boundaries (scope and resource wise), unable to influence or access what lies beyond their view.

## What is UnionFS

The Union File System (UnionFS) in Docker is a type of filesystem that allows multiple layers `to be stacked` on top of each other while appearing as a single unified filesystem. It is a key component of how Docker images and containers work, enabling efficient storage, reusability, and fast builds.

### How UnionFS Works in Docker

Docker images consist of multiple layers, each representing changes made at different points in time. When you create a new Docker container, the Union File System combines these layers into a single view. The major benefits of this approach include:

- Layering: Each instruction in a Dockerfile (such as RUN, COPY, ADD) creates a new layer. Layers are **read-only** except for the topmost container layer, which is writable.
- Reusability: Since layers are immutable, they can be reused by multiple containers, reducing storage usage and speeding up builds.
- Copy-on-Write (COW): When a container modifies a file, Docker does not change the original layer but instead copies the file to the writable container layer, ensuring efficiency and preserving the original image.
  
Basically, a layer, or image layer is a change on an image, or an intermediate image. Every command you specify (FROM, RUN, COPY, etc.) in your Dockerfile causes the previous image to change, thus creating a new layer. You can think of it as staging changes when you're using git: You add a file's change, then another one, then another one...

Consider the following Dockerfile:

```
FROM rails:onbuild
ENV RAILS_ENV production
ENTRYPOINT ["bundle", "exec", "puma"]
```

First, we choose a starting image: rails:onbuild, which in turn has many layers. We add another layer on top of our starting image, setting the environment variable RAILS_ENV with the ENV command. Then, we tell docker to run bundle exec puma (which boots up the rails server). That's another layer.

The concept of layers comes in handy at the time of building images. Because layers are intermediate images, if you make a change to your Dockerfile, docker will rebuild only the layer that was changed and the ones after that. This is called layer caching.