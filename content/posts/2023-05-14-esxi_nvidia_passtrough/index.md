---
title: VMWare ESXI Ubuntu Server 20.04.03 with RTX3070 
date: 2023-05-14T08:39:00+01:00
tags: 
    - esxi
---

# Configure nVidia GTX CUDA with passtrough in VMWare vCenter on Ubuntu 22

## TL;DR
Disable nouveau & enable unsuported GPU’s for open source drivers
```shell
sudo echo "blacklist nouveau" >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf  
sudo echo "options nouveau modeset=0" >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf  
sudo echo "options nvidia NVreg_OpenRmEnableUnsupportedGpus=1" >> /etc/modprobe.d/nvidia.conf

sudo update-initramfs -u  
sudo reboot
```

Install the drivers
```shell
wget https://download.nvidia.com/XFree86/Linux-x86_64/525.89.02/NVIDIA-Linux-x86_64-525.89.02.run
sudo chmod u+x NVIDIA-Linux-x86_64-525.89.02.run
sudo apt install build-essential
sudo apt install pkg-config libglvnd-dev
sudo ./NVIDIA-Linux-x86_64-525.89.02.run  -m=kernel-open
```

Then reboot, and install cuda without the drivers
```shell
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.0-1_all.deb
sudo dpkg -i cuda-keyring_1.0-1_all.de
bsudo apt-get update
sudo apt-get install cuda-toolkit-12-0 nvidia-cuda-toolkit
```


