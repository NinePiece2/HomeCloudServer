# Home Cloud Server

## Table of Contents
- [Home Cloud Server](#home-cloud-server)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Past Configurations and Insperations](#past-configurations-and-insperations)
    - [v1.0](#v10)
  - [Current Configurations](#current-configurations)
    - [TrueNAS](#truenas)
    - [Proxmox](#proxmox)
    - [Networking](#networking)
      - [HAProxy Load Blancer](#haproxy-load-blancer)
    - [Kubernetes](#kubernetes)
    - [Docker](#docker)
    - [DNS (LanCache and PiHole)](#dns-lancache-and-pihole)
  - [Future Upgradability](#future-upgradability)

## Introduction

First off most of this project will be documented in this README markdown along with the history of the project, and much of the troubleshooting done to get the server to where it is now.

Old versions of this documnet are available in the [Old Version 📁](/Old_Version)

## Past Configurations and Insperations

To begin with this project started for me in 2017 when I first heard about FreeNAS through learning how to build computers on YouTube. Like most who are uneducated about the server space I thought it was an extremely complicated mechanism that would be extremely difficult to understand. Now I know that a server is just a regular computer with different hardware requirements running a different Operating System. So began my first FreeNAS server on an old laptop with a single 160 GB Hard Drive and a bootable USB. This system worked for what it was but was limited by the single 100 mbps ethernet port on the old laptop and was only used for archival data.

It would eventually be upgraded to a 1TB Hard Drive and a 250 GB Boot SSD that would allow for more features to be used in newly released TrueNAS Core such as a plex media server. However, without any hardware redundancy and a cheap drive this NAS' storage drive would eventually become corrupted and all the data on it lost.

### v1.0

With the loss of data in mind as well as possible virtualization it was important to start with system that would not be easily corrupted. This means some sort of hardware redundancy, some of the options I considered were Raid 1, Raid Z and Raid Z2. Eventually I settled on using 3 4 Tb drives in Raid Z1 which would allow for one drive to fail completely without any data loss.

When it comes to the other hardware I just tried to reuse as much previous hardware as I could. This resulted in using a Ryzen 7 2700x with 32GB of ECC RAM.

## Current Configurations

There are two servers in the 'datacenter'.

### TrueNAS

After using the v1.0 configuration for 2 years it was clear that some improvements could be made to help with scailability, usability and reliability. To asist in this the server was upgraded to AMD Epyc running a 7551P processor with 128GB of RAM. The storage configuration stayed the same reusing the old 2 x 3 wide RAIDZ1 x 4TB drives and adding 2 more VDEVs to increase the storage space. The increased memory would allow the server to have 128TB of space using the ZFS recommended 1GB of RAM per TB of storage and until that amount of stoage was added the RAM could be used for virtualization which will be discussed more in detail in a later section.

Other than the aformemtioned 12 drive HDD array there is a 2 250GB SSD mirror for boot and a 3 250GB ssd RaidZ1 for Virtual Machine ZVol storage and for docker storage. There is also a single Kioxia CD6-R 3.84TB drive without any redundency which is used by a [LanCache](#dns-lancache-and-pihole) VM for the cache storage.

This server runs TrueNAS Scale ElectricEel.

Connectivity wise this server has an Nvidia P2000 for plex media encoding, an LSI 9207-8I along with the 8 onboard SATA to connect 16 drives as well as an LSI 9200-8e connected to a 45 Bay Supermicro JBOD I got off Ebay. There is also a Mellanox ConnectX-3 CX354A Dual 40GbE QSFP+ network card.

[<img src=images/CurrentConfig.PNG height=500>](images/CurrentConfig.PNG)

### Proxmox

After using Proxmox 7 in a virtual machine in the original [v1.0](#v10) configuration I decided to use the old components from v1.0 and run Proxmox on baremetal. The server reused the Ryzen 7 2700x with 32GB of ECC RAM and added 2 500GB SSDs as a mirrored boot device. Proxmox allows us to use this storage for VMs and LXCs so this was enough for me presently. This server also has a dual port 10G NIC that is link aggrigated for 20G using LACP.

This proxmox server runs multiple Virtual Machines and Linux Containers, the virtual machines include a Windows MineCraft Server VM that manages backups, server restarts and updates. The Linux Containers (LXCs) run some docker based game servers such as CS:2 and Garry's Mod and more importantly host multiple services that I have created, for more information visit my [Portfolio Site 💼](https://romitsagu.com/projects). The LXCs that are associated with websites also run self-hosted GitHub action runners that allow for Continous Deployment changes to be made when a new docker image of the site is available.

### Networking

When it comes to neworking the setup is not too advanced, I have a Ubiquiti Dream Machine Pro Max (UDM-Pro-Max) that handles routing and the subnet that is setup for the data center as well as setting clients to use the [local DNS](#dns-lancache-and-pihole). There is also a Brocade ICX6610-48-E that is connected to the UDM at 10G using a diract attach cable that I got from FS.com. The Brocade switch has all of the full licenses unlocked following a guide from [Fohdeesha](https://fohdeesha.com/docs/brocade-overview.html). Due to the licenses the switch has 48 1G RJ45 ports, 8 10G SFP+ ports, 8 10G SFP+ ports that come from two 40G QSFP+ to 4x10G DAC cables and 2 other 40G QSFP+ ports. Making this switch extremly versitle and cost efficient.

These devices have allowed me to connect the servers at extremly high speeds and ensure that they can communicate quickly with eachother as well as clients on and off the local network.

#### HAProxy Load Blancer
To manage inbound connections requesting different websites that run on the lcoal network I have a HAProxy Load Balancer that takes in requests over port 443 and routes them to the correct local webserver and encrypts traffic using SSL through Cloudflare or LetsEncrypt depending on the service's needs.

### Kubernetes

Spread across the two servers are 3 MicroK8s Kubernetes Nodes with the primary node on a Virual Machine on the [TrueNAS](#truenas) server due to the large amount of RAM and 2 on the [Proxmox](#proxmox) server that function as backups for if something goes wrong with the primary node.

Kubernetes Pods:
[<img src=images/KubernetesPods.png >](images/KubernetesPods.png)

### Docker

There are docker containers that run on both servers as shown in the images below using the [TrueNAS](#truenas) GUI and Portainer for the [Proxmox](#proxmox) server. For example, the [TrueNAS](#truenas) server runs things like Plex Media Server and Immich which take up alot of storage space for data and the [Proxmox](#proxmox) server hosts a Microsoft SQL Server 2022 Express container.

[TrueNAS](#truenas) Server:<br/>
[<img src=images/TrueNASApps.png height=500>](images/TrueNASApps.png)

[Proxmox](#proxmox) Server:<br/>
[<img src=images/ProxmoxPortainer.png >](images/ProxmoxPortainer.png)

### DNS (LanCache and PiHole)

I have LanCache and PiHole instances that run off the [TrueNAS](#truenas) server, the PiHole DNS server runs on docker and the LanCache DNS runs off a VM that also has a Kioxia CD6-R 3.84TB drive to use for caching data.

To better visualize this data I have modified an existing LanCache UI project and made changes to include more uselful features for myself. Details are available [here](https://github.com/NinePiece2/NineLanCacheUI).

## Future Upgradability

The next likely upgrades are to add more drives to the [TrueNAS](#truenas) server using the JBOD as well as adding more nodes to the [Proxmox](#proxmox) server to allow for high availabilty of Virtual Machines and Linux Containers.