---
title: Configure PXE Booting on pfSense
categories: [blog]                   
tags: [linux]                   
featured: false
layout: blog/post
img_folder: '/images/posts/pxe-pfsense'  
img_cover: '15285.jpg'
author: totomz

# TODO
#keywords: [pxe, pfsense, "boot from lan", "pxe boot"]
---

Nowadays is pretty uncommon to setup a workstation/laptop, and installing a Linux distro using a CD is even less frequent. Today we had to 
setup a workstation that we use to run some tests - we need a a bare metal server for these tests - and none of us had an USB stick, neither a DVD Writer :)

How to install an operating system in the era of cloud? Easily, with PXE and pfSense!  

{% include blog/workflows/pxe-pfsense.html %}

{% include blog/figure.html img=page.img_cover alt="How to configure pfSense as a PXE boot server" caption="" %}

{: #pxe}
# What is PXE? How does boot from LAN actually works?
{% lipsum 4 %}

# Configure the Workstation
This step is easy: simply configure the workstation to boot from LAN!

{: #setup-pfsense-dhcp-tftp}
# Setup pfSense
[pfSense](https://www.pfsense.org/) is an open source firewall/router computer software distribution based on FreeBSD. It's functionalities can be extended thanks to its plugin system

To use our pfSense box as a boot server, we need to
* install a tftp server;
* deploy a netboot image and ramdisk;
* configure the DHCP server;

No worries, it is **easier** that it seems!

## Install a tftp server
Open the pfSense admin dashboard and navigate to System -> Package Manager

In the `Available packages` menu, search  `tftp`.

{% include blog/figure.html img="pfsense_install_tftp.png" alt="Install TFTP server package in pfSense" caption="Search and intstall the TFTP package" %}

Next, refresh the page and enable the TFTP server: Select `Service` -> `TFTP Server` and check the `Enable TFTP Service` checkbox. 

{: #deploy-netboot-image }
## Upload a NetBoot image
We choosed to install Ubuntu 18.04. Find the URL where to download the netboot iso. In our case, the URL is http://archive.ubuntu.com/ubuntu/dists/bionic-updates/main/installer-amd64/current/images/netboot/
and the image name is `netboot.tar.gz`.

The image must be uploaded in pfSense. Download the tar acrhive and uncompress it locally. Now you have to copy the contents o the tar archive in the root of yout tftp server. 
A simple way is to SSH in the pfSense box and to download and extract the tar archive directly in the `/tftpboot` directory.


## Configure the DHCP 
Set in the DHCP options the TFTP server and the image to use to boot theservers. Select  `Services` -> `DHCP Server` and scroll at the bottom of the page.

Set the following configurations:
* TFTP Server **AND** Next server: the ip address of the pfSense box;
* Default BIOS file name: the image youwant to use, `pxelinux.0` in our case;
* Enable the `Enable network booting` checkbox;

{: #done }
# Conclusions
That's it! Now boot your workstation, and install the operating system!