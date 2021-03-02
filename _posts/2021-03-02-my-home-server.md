---
title: "My (little) home server"
categories:
  - Hardware
tags:
  - hardware
---

In Sept 2020 I decided that I needed a home server to do some development.
For the purpose I wanted a system able to execute a decently-sized Apache Spark job, and run some middle-tier TensorFlow model.
The server is currently dual-booting Ubuntu and Win10.

List of component:

* AMD Ryzen 9 3900X 12-core, 24-thread ![right-aligned-image](/images/amd-ryzen-9.png){: .align-right}
* Sabrent 1TB Rocket NVMe 4.0 Gen4 PCIe M.2 Internal SSD
* EVGA GeForce RTX 2060 KO GAMING 6GB Video Card
* G.SKILL Ripjaws V Series 64GB (2 x 32GB) 288-Pin DDR4 SDRAM DDR4 3200 (PC4 25600)
* ASRock X570M PRO4 AM4 AMD X570 SATA 6Gb/s Micro ATX AMD
* Western Digital 4TB WD Blue PC Hard Drive - 5400 RPM

The basic idea is 24 threads, 64 GB fast/sizable memory and super fast 1TB drive, plus a GPU for CUDA operations is the minimum setup needed to run decent jobs.
I can infer the embeddings of the full wiki english dataset pretty quickly :D
I also use a slow Western Digital disk to store big parquet/delta lake datasets.
