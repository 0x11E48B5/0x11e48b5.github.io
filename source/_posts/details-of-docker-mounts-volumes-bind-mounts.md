---
title: Understanding docker mounts
date: 2025-02-26 15:39:12
tags: ["devops", "docker", "container"]
---

# What is the meaning of Mount?

If you have used Docker before, which I’m sure you have, you must have realized that data written to the container file system is completely lost when your container has been accidentally or intentionally restarted. This presents a significant problem since we, as engineers, bear the heavy responsibility of ensuring **persistent** application data. To address this issue, Docker has provided several ways called Mounts to overcome this challenge and keep data on a more permanent basis.

As of today, there are three types of **mounts** in Docker, each designed for specific reasons and use cases. These types of mounts are:

- Bind mounts
- Volume mounts
- Tmpfs mounts

## What is bind mount?

By default host directories are not available in the container file system , but with bind mounts we can access the host filesystem. bind mount is a way to connect or link a directory or file from your computer’s file system to a specific location inside a Docker container.

You can easily update the files on your computer, and the changes will be instantly reflected inside the container without the need to rebuild or modify the container itself.

Bind mounts tightly couple the container to the host machine’s filesystem, which means that processes running in a container can modify the host filesystem. This includes creating, modifying, or deleting system files or directories. Therefore, it is crucial to be cautious **with permissions** and ensure proper access controls to prevent any **security risks** or conflicts. This mounting type is not controlled from docker.

- Maps a specific directory on the host to a container
- Allows direct access to host files
- Less portable than volumes (depends on host filesystem)

{% asset_img bind-mount.webp %}

In docker, you can use Bind mount in CLI like this;

```
docker run -v /host/path:/path/in/container image:tag
```

In Docker Compose;

```
services: 
    app: 
      image: image:tag 
      volumes: 
        - /host/path:/path/in/container 
```

## What is volume mount?

Volume mounts in Docker provide a way to persist and share data between containers and the host system. They allow containers to access and store data beyond their lifecycle, ensuring that data is not lost when a container stops or is removed.

Volume mounts are like bind mounts but they are fully managed by docker itself . Volumes are stored in the Linux VM rather than the host, which means that the reads and writes have much lower latency and higher throughput.

volume mounts are a way to manage and store data separately from the containers themselves. They provide a means to **store** and **share** files and directories that can be persist even if containers are **deleted** or **recreated.** If we want to look at benefits of volume mounts;

- Stored in /var/lib/docker/volumes/
- Managed entirely by Docker
- Ideal for persistent and shareable data

Using example in CLI;

```
docker volume create my_volume
docker run -d -v my_volume:/data --name my_container ubuntu
```

## Ephemeral container filesystem - tmpfs

In the context of Docker, tmpfs mounts allow you to mount a temporary file system into a container's filesystem, which `resides in memory` rather than on disk. This can be useful for scenarios where you need a filesystem that is fast and volatile, such as storing temporary files or caches.

Using example in CLI;

```
docker run --tmpfs /path/in/container image:tag
```

another example with settings of size and mode

```
docker run -d --tmpfs /mytmpfs:size=100m,mode=1777 ubuntu
```
If you're using Docker compose for deploying containers, you can use tmpfs like this;

```
version: "3.9"
services:
  app:
    image: ubuntu
    tmpfs:
      - /mytmpfs:size=200m
```

## Useful volumes commands to manage

- To list all volumes
```
docker volume ls
```
- To inspect details of specific volume use inspect
```
docker volume inspect volume_name
```
- To remove volume
```
docker volume rm volume_name
```
- To clean up unused volumes
```
docker volume prune
```

## Comparing tmpfs vs Volumes vs Bind Mounts

| Feature  | tmpfs Mount | Volume | Bind Mount |
|----------|------------|--------|------------|
| **Persists after container restart?** | ❌ No | ✅ Yes | ✅ Yes |
| **Stored in RAM?** | ✅ Yes | ❌ No | ❌ No |
| **Performance** | ⚡ Fastest | 🚀 Fast | 🐢 Depends on disk |
| **Use Case** | Temporary, sensitive, or high-speed data | Persistent app data | Share host files with a container |

## Conclusion: Docker Volumes and Data Persistence

**Docker volumes** are the recommended approach for `data persistence` in containers. Unlike bind mounts, which rely on the host filesystem structure, volumes are fully managed by Docker, making them more portable, secure, and efficient for storing application data.

Since container filesystems are ephemeral, using volumes ensures that critical data—such as databases, logs, and configuration files—remains intact even if the container is stopped or removed. For optimal performance and security, choose volumes for long-term storage, bind mounts for host-container file sharing, and tmpfs mounts for temporary, high-speed data storage in RAM.