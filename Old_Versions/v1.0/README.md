# Home Cloud Server

## Table of Contents
- [Home Cloud Server](#home-cloud-server)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Past Configurations and Insperations](#past-configurations-and-insperations)
  - [Current Configurations](#current-configurations)
  - [Virtual Environments](#virtual-environments)
  - [Setting Up a Pair of Mellanox Connectx-3](#setting-up-a-pair-of-mellanox-connectx-3)
    - [Windows Setup](#windows-setup)
    - [TrueNAS Scale Setup](#truenas-scale-setup)
  - [Apps/Docker and Kubernetes](#appsdocker-and-kubernetes)
  - [Modifications](#modifications)

## Introduction

First off most of this project will be documented in this README markdown along with the history of the project, and much of the troubleshooting done to get the server to where it is now.

## Past Configurations and Insperations

To begin with this project started for me in 2017 when I first heard about FreeNAS through learning how to build computers on YouTube. Like most who are uneducated about the server space I thought it was an extremely complicated mechanism that would be extremely difficult to understand. Now I know that a server is just a regular computer with different hardware requirements running a different Operating System. So began my first FreeNAS server on an old laptop with a single 160 GB Hard Drive and a bootable USB. This system worked for what it was but was limited by the single 100 mbps ethernet port on the old laptop and was only used for archival data.

It would eventually be upgraded to a 1TB Hard Drive and a 250 GB Boot SSD that would allow for more features to be used in newly released TrueNAS Core such as a plex media server. However, without any hardware redundancy and a cheap drive this NAS' storage drive would eventually become corrupted and all the data on it lost.

## Current Configurations

With the loss of data in mind as well as possible virtualization it was important to start with system that would not be easily corrupted. This means some sort of hardware redundancy, some of the options I considered were Raid 1, Raid Z and Raid Z2. Eventually I settled on using 3 4 Tb drives in Raid Z1 which would allow for one drive to fail completely without any data loss.

When it comes to the other hardware I just tried to reuse as much previous hardware as I could and the current hardware configuration is shown in the image below:

<br/>

[<img src=images/CurrentConfig.PNG height=500>](images/CurrentConfig.PNG)

## Virtual Environments

The software setup is quite simple, the server runs bare metal TrueNAS Scale Bluefin and using the built in hypervisor there are two virtual machines. As a side note the virtual machines use a bridged network interface so they can still communicate with the main TrueNAS server to get data from its SMB shares. The first is an ubuntu server VM that currently hosts a Minecraft and Assetto Corsa Servers and the second is a Proxmox VM that currently only hosts a Kemp Load Balancer on the Proxmox hypervisor. The load balancer lets me setup a DNS for all of the pages so that I can access them from anywhere around the world and they all have SSL certificates. The Kemp Load Balancer setup came from [here](https://www.youtube.com/watch?v=LlbTSfc4biw&ab_channel=NetworkChuck) however some modifications were needed as I was using Proxmox and not ESXI. All that was needed was a conversion from vmdk to qcow2 which can be done using the following code:

```
qemu-img convert -f vmdk filename.vmdk -O qcow2 filename.qcow2
```

As a note because Proxmox was not running on bare metal KVM hardware virtualization needed to be turned off:

[<img src=images/KVM.PNG height=400>](images/KVM.PNG)

After this the system would boot loop after reaching the login screen. This was fixed by changing the SCSI Controller to VMware PVSCSI in the hardware tab:

[<img src=images/SCSIController.PNG height=400>](images/SCSIController.PNG)

After this its as simple as following the rest of the YouTube video and setting everything up through Cloudflare. The load balancer is currently hosts the TrueNAS server on the domain [truenas.romitsagu.com](https://truenas.romitsagu.com).

For my application that meant having a single Virtual IP Address with multiple Sub Virtual Services that each have one Real Server associated with them and one Content Matching Rule. The images below show the Sub Virtual Services and the Content Matching Rules that were created:

[<img src=images/SubVirtualServices.png height=220>](images/SubVirtualServices.png)

Sub Virtual Services

[<img src=images/ContentMatchingRules.png height=170>](images/ContentMatchingRules.png)

Content Matching Rules

## Setting Up a Pair of Mellanox Connectx-3

This honestly should have been simpler but took a lot of time to figure out as there are not many people running this configuration. I bought 2 Mellanox ConnectX-3 CX354A Dual 40GbE QSFP cards that had been flashed to MCX354A-FCBT which allowed it to function in Ethernet mode as well as InfiniBand mode. One is installed in my computer while the other is installed in the server and they are connected by a direct attach copper (DAC) 40 GbE cable. The idea to do this came to me after watching this [video](https://www.youtube.com/watch?v=OZ8ZS1CwbPI&ab_channel=RaidOwl).

### Windows Setup

The setup on Windows 10 was very simple after getting the divers installed from [here](https://network.nvidia.com/products/adapter-software/ethernet/windows/winof-2/). Then going to device manager -> System Devices -> Mellanox ConnectX-3 VPI Network Adapter -> Properties -> Port Protocol and setting both ports to ETH for ethernet mode:

[<img src=images/MellanoxWindows.PNG height=400>](images/MellanoxWindows.PNG)

In the image both settings are grayed out as I set it up using Mellanox MFT which was unnecessary and done as a troubleshooting step. However, if you go to the options and see the option is grayed out download MFT from [here]( https://network.nvidia.com/products/adapter-software/firmware-tools/) and use [this]( https://blog.insanegenius.com/2020/06/21/moving-from-unraid-to-proxmox-ve/) guide to help you set it to Ethernet mode using MFT.

### TrueNAS Scale Setup

For this Operating System the setup was not as straight forward there are 2 main problems compared to the Windows setup. The first is that there is no way to change the ports on the card to Ethernet mode using a GUI so MFT needs to be used and the second problem is that the Kernal does not allocate memory for the driver by default.

To solve the first problem, we need to connect to the TrueNAS server's shell. This can be done through the web GUI or enabling SSH to connect from another machine. Then create a folder somewhere (I did it my user's home directory) using the command line and use wget to download MFT from [here]( https://network.nvidia.com/products/adapter-software/firmware-tools/) by selecting version 4.22.1-LTS -> Linux -> DEB based and x64 or whatever architecture you are using then right click on the link and copying it so you can use ``` wget link.com ``` to download it and then unzip it in that folder. From here you can follow the rest of the guide from [here](https://blog.insanegenius.com/2020/06/21/moving-from-unraid-to-proxmox-ve/). At this point you should have the device running in Ethernet mode.

To solve the second problem we once again need to connect to the TrueNAS server's shell as mentioned above. Once in the terminal enter the command ```nano /usr/local/bin/truenas-grub.py ```. Then from here scroll down to ```config  = {``` and make sure that it is equal to:
```
config = [
        'GRUB_DISTRIBUTOR="TrueNAS Scale"',
        'GRUB_CMDLINE_LINUX_DEFAULT="pci=realloc=off libata.allow_tpm=1 systemd.unified_cgroup_hierarchy=0 amd_iommu=on iommu=pt '
        'kvm_amd.npt=1 kvm_amd.avic=1 intel_iommu=on zfsforce=1'
        f'{f" {kernel_extra_args}" if kernel_extra_args else ""}"',
    ]
```
Then restart the system and the card should show up in the TrueNAS dashbord like this:

[<img src=images/MellanoxNotConnected.PNG height=400>](images/MellanoxNotConnected.PNG)

The next part is simple we just need to setup IP addresses on the device as the two systems are connected directly without a switch to automatically assign them. On Windows go to Network Connections and right click on the network adapter that the cable was attached to and select properties. Then double click on the Internet Protocol Version 4 and enable the option to use a given address and to use a given DNS server address. Enter in an IP address and Subnet Mask as shown in the image below.

[<img src=images/WindowsIPConfig.PNG height=500>](images/WindowsIPConfig.PNG)

Now on the TrueNAS side from the web GUI go to network -> then select the interface that the cable is connected to and add an Alias. The IP address used here should have the same staring portions with the last part changed and they should use the same subnet mask.

[<img src=images/TrueNASIPConfig.PNG height=400>](images/TrueNASIPConfig.PNG)

## Apps/Docker and Kubernetes

After all of this setup I have a few apps that run off the built in Docker and Kubernetes in TrueNAS Scale. From the Official TrueNAS charts I have a Plex Media Server that has media that I have collected through the years and want to preserve digitally. I also have netdata which allows me to track the usage of the server overall as well as any apps that are running off of it. I used to also have a Pi-Hole server that ran as an app form the Truecharts catalog but it stopped working so I just turned it off and haven’t taken the time to fix it.

[<img src=images/Apps.PNG height=200>](images/Apps.PNG)

</br>

When it comes to Docker and Kubernetes there is a third additional virtual machine that was added to run Microk8s. The plan was to originally just use Kubernetes however due to a limitation of Ubuntu 22.04 it was not initializing. However, Microk8s can be installed in the installation of Ubuntu 22.04 so that is what was chosen. Microk8s acts exactly like Kubernetes just having one application that handles, Kubectl, Kubelet and Kubeadm. Currently the addons that have been enabled:

[<img src=images/Microk8s_Addons.png height=150>](images/Microk8s_Addons.png)

</br>

Currently running on Kubernetes in this virtual machine is:
- [ESPN Betting Website Prototype (Full Stack)](https://github.com/NinePiece2/ESPN-Betting-Website-Prototype-Full-Stack)
- [ArtiFace: Facial Art Synthesizer](https://github.com/Siddhant0701/ArtiFace)

These are the pods for those projects:

[<img src=images/Pods.png height=150>](images/Pods.png)

## Modifications

After the initial setup another 3 drive Vdev was added and striped to the already exisiting Raid-Z1 pool.
