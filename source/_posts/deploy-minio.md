---
title: Setting Up MinIO in Multi-Node Mode for Redundant and Scalable Storage
date: 2025-03-24 16:30:10
tags: ["devops", "minio", "architecture", "object storage"]
---

# What is MinIO?

MinIO is a high-performance, S3-compatible `object storage solution` designed for private cloud, hybrid cloud, and edge computing environments. It provides a scalable, distributed architecture for storing large amounts of unstructured data such as images, videos, backups, and log files.

MinIO was designed from the beginning to be a fully compatible alternative to Amazon’s S3 storage API. They claim to be the most compatible S3 alternative while also providing comparable performance and scalability MinIO also **provides a variety of deployment** options. It can run as a `native application` on most popular architectures and can also be deployed as a `containerized application` using Docker or Kubernetes.

Additionally, MinIO is open-source software. Organizations are free to use it under the terms of the AGPLv3 license. Just beware that this option comes with no support aside from online documentation and the MinIO user community. For larger enterprises, paid subscriptions with dedicated support are also available.

## What is Object Storage

Object storage is a data storage architecture that manages data as objects rather than as files (file storage) or blocks (block storage). It is designed for scalability, durability, and easy retrieval, making it ideal for storing large amounts of unstructured data like images, videos, backups, logs, and analytics data.

The concept of object storage is similar to that of a standard Unix file system, but instead of directories and files, we use buckets and objects. In traditional file system, you can think `directories as buckets` and `files` inside of those directories `as objects.`

{% asset_img block-order.jpg %}

## MinIO Deployment Types

There are 3 types of deployment types available for MinIO;

- **Single Node Single Drive (SNSD)**
  - SNSD deployments use a zero-parity erasure coded backend that provides no added reliability or availability beyond what the underlying storage volume implements. These deployments are best suited for local testing and evaluation, or for small-scale data workloads that do not have availability or performance requirements.
- **Single Node Multi Drive (SNMD)**
  - SNMD deployments provide drive-level reliability and failover/recovery with performance and scaling limitations imposed by the single node.
- **Multi Node Multi Drive (MNMD) - For Production**
  - MNMD deployments support erasure coding configurations which tolerate the loss of up to half the nodes or drives in the deployment while continuing to serve read operations. Use the MinIO Erasure Code Calculator when planning and designing your MinIO deployment to explore the effect of erasure code settings on your intended topology.


### General Prerequisites for Better Performance

1. **Use Local Storage**
   - Direct-Attached Storage (DAS) has significant performance and consistency advantages over networked storage (NAS, SAN, NFS). MinIO strongly recommends flash storage (NVMe, SSD) for primary or “hot” data.
2. **Use XFS-Formatting for Drives**
	- MinIO strongly recommends provisioning XFS formatted drives for storage. MinIO uses XFS as part of internal testing and validation suites, providing additional confidence in performance and behavior at all scales. MinIO does not test nor recommend any other filesystem, such as EXT4, BTRFS, or ZFS.
3. **Use Consistent Type of Drive**
	- MinIO does not distinguish drive types and does not benefit from mixed storage types. Each pool must use the same type (NVMe, SSD). For example, deploy a pool consisting of only NVMe drives. If you deploy some drives as SSD or HDD, MinIO treats those drives identically to the NVMe drives. This can result in performance issues, as some drives have differing or worse read/write characteristics and cannot respond at the same rate as the NVMe drives.

> MinIO recommends a minimum of 32GiB of memory per host. See Memory for more guidance on memory allocation in MinIO.
	
## Install MinIO

In this tutorial we will teach how to install minIO for Ubuntu/Linux systems. Use one of the following options to download the MinIO server installation file for a machine running Linux on an Intel or AMD 64-bit processor.

### Install the MinIO Binary on Each Node

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20250228095516.0.0_amd64.deb -O minio.deb

sudo dpkg -i minio.deb
```

### Create folders and mount disks to these folders

In this example i'll create 4 folders in "/mnt" folder. Also i've 4 seperate NVMe SSD Disks that attached to my machine. 

> MinIO recommends at least 4 disks per node for distributed deployments. This is because MinIO uses erasure coding to ensure data redundancy and availability, and it requires a minimum of 4 drives to effectively distribute and protect data.

```bash
mkdir /mnt/disk1 /mnt/disk2 /mnt/disk3 /mnt/disk4
```

### Create /etc/fstab to mount disks persistently
Before adding, we recommend you to format you disks in XFS format. Here is an example for one disk. Do this steps for each disks.

```bash
mkfs.xfs /dev/sdb -L DISK1
```
You should add your disks like below.

```bash
# /etc/fstab: static file system information.
LABEL=DISK1     /mnt/disk1      xfs     defaults,noatime 0 2
LABEL=DISK2     /mnt/disk2      xfs     defaults,noatime 0 2
LABEL=DISK3     /mnt/disk3      xfs     defaults,noatime 0 2
LABEL=DISK4     /mnt/disk4      xfs     defaults,noatime 0 2
```
### Give local dns name for nodes IPs

Open your `/etc/hosts` folder and write like this.

```bash
192.168.10.10 minio1.<your-org-name>.net
192.168.10.11 minio2.<your-org-name>.net
```

### Create folder in all disks with minio name

```bash
mkdir -p /mnt/disk1/minio
mkdir -p /mnt/disk2/minio
mkdir -p /mnt/disk3/minio
mkdir -p /mnt/disk4/minio
```

### Create systemd service

The `.deb` or `.rpm` packages install the following systemd service file to /usr/lib/systemd/system/minio.service. For binary installations, create this file manually on all MinIO hosts.

```bash
[Unit]
Description=MinIO
Documentation=https://min.io/docs/minio/linux/index.html
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local

User=minio-user
Group=minio-user
ProtectProc=invisible

EnvironmentFile=-/etc/default/minio
ExecStartPre=/bin/bash -c "if [ -z \"${MINIO_VOLUMES}\" ]; then echo \"Variable MINIO_VOLUMES not set in /etc/default/minio\"; exit 1; fi"
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# MinIO RELEASE.2023-05-04T21-44-30Z adds support for Type=notify (https://www.freedesktop.org/software/systemd/man/systemd.service.html#Type=)
# This may improve systemctl setups where other services use `After=minio.server`
# Uncomment the line to enable the functionality
# Type=notify

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Specifies the maximum number of threads this process can create
TasksMax=infinity

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target

# Built for ${project.name}-${project.version} (${project.name})
```

The minio.service file runs as the minio-user User and Group by default. You can create the user and group using the groupadd and useradd commands. 

The following example creates the user, group, and sets permissions to access the folder paths intended for use by MinIO. These commands typically require root (sudo) permissions.

```bash
groupadd -r minio-user
useradd -M -r -g minio-user minio-user
chown -R minio-user:minio-user /mnt/disk1 /mnt/disk2 /mnt/disk3 /mnt/disk4
```

### Create the Service Environment File

Create an environment file at `/etc/default/minio`. The MinIO service uses this file as the source of all environment variables used by MinIO and the minio.service file.

The following examples assumes that:
- You have 2 successfully installed ubuntu node.
- All hosts have four locally-attached drives with sequential mount-points:

```bash
# Set the hosts and volumes MinIO uses at startup
# The command uses MinIO expansion notation {x...y} to denote a
# sequential series.
#
# The following example covers four MinIO hosts
# with 4 drives each at the specified hostname and drive locations.
# The command includes the port that each MinIO server listens on
# (default 9000)

MINIO_VOLUMES="https://minio{1...2}.example.net:9000/mnt/disk{1...4}/minio"

# Set all MinIO server options
#
# The following explicitly sets the MinIO Console listen address to
# port 9001 on all network interfaces. The default behavior is dynamic
# port selection.

MINIO_OPTS="--console-address :9001"

# Set the root username. This user has unrestricted permissions to
# perform S3 and administrative API operations on any resource in the
# deployment.
#
# Defer to your organizations requirements for superadmin user name.

MINIO_ROOT_USER=minioadmin

# Set the root password
#
# Use a long, random, unique string that meets your organizations
# requirements for passwords.

MINIO_ROOT_PASSWORD=CHANGEME
```

### Enable and start service

After that we need to enable and start our service.

```bash
sudo systemctl enable minio.service
sudo systemctl start minio.service
```

# Final Thoughts

MinIO is a powerful, high-performance, and scalable object storage solution that seamlessly integrates into modern cloud-native and on-premises environments. Whether you're looking for a simple standalone setup for development or a highly available, distributed storage cluster for production, MinIO provides the flexibility to meet your needs. Its S3-compatible API, lightweight architecture, and ability to run on commodity hardware make it a compelling choice for organizations of all sizes.

As data continues to grow exponentially, having a reliable and scalable object storage solution is more important than ever. MinIO not only simplifies storage management but also ensures high availability, data integrity, and security. If you’re looking for an efficient, open-source alternative to traditional cloud storage, MinIO is definitely worth considering.
