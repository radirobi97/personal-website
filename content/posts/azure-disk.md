+++
author = "Robert Radi"
title = "Everything you should know about Azure Managed Disks"
date = "2022-01-11"
description = "This article gives an in-depth guide to Azure Managed Disks."
tags = [
    "azure",
    "disk",
]
categories = [
    "azure",
    "in-depth",
]
series = ["Themes Guide"]
aliases = ["migrate-from-jekyl"]
+++

One way to store data in Azure is using Azure Managed disks. Furthermore, disks will host your operating system. In this article, I am trying to give an in-depth look into Azure Disks. 

## Basics

* disks are independent first-class resources, meaning they can live on their own. This was not always the case, because at the beginning disks were a part of storage accounts. So, limits of storage accounts had to be taken into consideration. To be completely correct, that is the case today as well, however it is abstracted from the end-customers. If you want to explore this, visit one of your disks, and choose disk export. The domain indicates that this is a storage account indeed under-the-hood. 
{{< figure src="/images/disk_export.png" title="Disk Export" >}}
* disks can take the advantage of using RBAC
* snapshot can be taken to provide consistent backups. These snapshots can be incremental or full.
* disks have their own limits for IOPS, throughput
* in Azure we always have to pay for the provisioned size and not for the used one
* disk size can not be decreased

## Characteristics of managed disks
* **capacity**: how much data can be stored on the disk. This size is defined in mebi-, gibi-, tebibytes, meaning we have to multiply the things by 1024.
* **IOPS**: Input Output Operations per Second, the number of input/output operations the disk can handle within one second. Every IO operand has its own size, that is the IO depth. More IO operands with a smaller IO depth is causing much more load then fewer IO operands with a larger IO depth. That is because every IO operand has its own overhead and the CPU should manage those. If we want to move a lot of small files then large IOPS is required. A good example for this is an OLTP application. 
* **throughput**: bandwidth is an alias. It defines the amount of data can be sent to the disk within one second. The relations ship between IOPS and throughput is the followin: `throughput = IOPS * IO_depth`. If we want to move a small large files then the throughput is critical. For example a data warehouse.
* **latency**: the time it takes to get a response from the disk for the given request. Most of the time read and write latency differs. The IO metric is useless without knowing the latency. Latency is critical for streaming applications. 

## Key indicators we should take into consideration before choosing a disk SKU
* How much data should be stored?
* What is the required latency?
* What are the average IO and throughput required by the application?
* What are the peak values?
   * What are the exact values of these peaks?
   * How long do they last?
   * How often do they occur?

## Disk types
1. Standard HDD
   * Performance is not guaranteed, we have only an "upto" value
   * Only locally redundant storage (LRS) is supported. LRS means we have 3 copies of the data. 
   * Performance can be increased only by increasing the disk size
2. Standard SSD
   * Performance is not guaranteed, we have only an "upto" value
   * Zone redundant storage (ZRS), LRS are supported. Zone redundant means that we have 3 copies of the data and each copy is stored in a different avaibility zone. 
   * Credit based bursting is available for free for IOPS and throughput
   * Performance can be increased only by increasing the disk size
3. Premium SSD
   * Performance is guaranteed
   * ZRS, LRS are supported
   * This is where enterprise grade disks begin
   * Performance tier can be increased **online (no need for detaching the disk from the VM)** without having to increase the disk size. Increasing the performance tier takes up to max 1 hour to take effect. We will be charged based on the following formula: `cost = max(PerformanceTIER, SizeTIER)`. Performance tier can be decreased as well. 
   * Under P30 tier credit based bursting is available. Above that we can take advantage of ondemand bursting which is **not free**.
   * They can be used as a shared disk, meaning a given disk can be attached to multiple VM at the same time. 
4. Ultra disk
   * Performance is guaranteed
   * ZRS, LRS are supported
   * IOPS and throughput can be finetuned separately
   * They can be used as a shared disk.
   * Ultra disk is a good choice for database storage.

# Disk SKUs and VM SKUs
Every type of disks has its own SKU. This SKU defines the capabilities and limits of a given disk. You can see one example on the following pictuce.
![disk-sku](/images/disk_sku.png)
Some explanations to the attributes:
* **Disk Size*: how much data can be stored on the disk
* **Price per month**: the monthly price of the disk
* **Max IOPS (Max IOPS w/ bursting)**: How many IOPS the disk can offer with and without bursing. Bursting discussed at a later section of this post.
* **Max throughput (Max throughput w/ bursting)**: How much throughput the dis can offer with and without bursting.
* **Price per mount per month (Shared Disk)**: Premium SSD and ultra disks can be attached to multiple VMs at the same time. We will pay this amount of money for each mount.

It is important to know that not only the disks but each VMs has its own limits and values for IOPS and throughput as well:
![vm-sku](/images/vm_sku.png)
A powerful disk is not worth anything without a proper VM. We should differentiate two different scenarios:
* When the VM could handle more IOPS/throughput than the disk can offer: we can say that the workload is IO/throughput capped on the disk level.
* When the disk offers more IOPS/throughput than the VM can handle: we can say that the workload is IO/throughput capped on the VM level. 
If we are working with network based storage, the network limits of the VM should be taken into consideration. 

# Host cache
Probably you have noticed on the previous picture that there is something called a cached or uncached IOPS/throughput. This has to do with the feature called "host caching". When you attach a disk to the VM, you have the option to define whether to use host caching on that particular disk.
![host-cache](/images/host_cache.png)
Host cache is all about bringing the data closer to the VM to be able to perform the reads, writes quicker. The documentation of every VM states how large this host cache could be. 

There are different types of host caching.
* **None**: no caching at all. Configure host-cache as None for write-only and write-heavy disks, for example when writing a huge amount of logs.
* **read-only**: reads are server from the cache. Use this for read heavy workloads.
* **read/write**: reads are served from the cache, and writes propagates to the disk asyncronously. Use only if your application properly handles writing cached data to persistent disks when needed. If the app does not handle it properly that could lead to data loss when the VM crashes.

But what is cached IOPS and uncached IOPS? For the sake of simplicity we will discuss IOPS but the same applies for throughput as well. How each IO operation count into these limits?

##### using read only host cache
![host-cache-limit](/images/host_cache_limits.png)
* First of all lets take a read operation. If the data is present in the cache then it will be read from the cache. It will count into cached limits of the VM. This is illustrated on the above picture by black arrows. If the data is not present in the cache, then first the data will be read from the disk into cache, and after that from the cache into the VM. So, this operation counts into the cached limits and into the uncached limits as well. 
* What about a write operation? It will also count into both of the limits due to the data first will be written into the cache, and after that from the cach into the disk.

##### using read/write host cache
* read operations are performed the same way just like using a read only cache
* write operations are different. The given write operation is marked as done when the data has been written to the cache, this counts into the cached limits. The data will be written into the disk asyncronously, and this counts into the uncached limits
