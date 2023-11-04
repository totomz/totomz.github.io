---
title: Private VPN with Taliscale and T
date: 2023-03-09T16:14:00+01:00
tags: 
    - aws
    - tailscale


---

Goal: build a low cost vpn concentrator?

First of all, we need an AMI id to deploy. We can use terraform to query AWS AMI. We want an Ubuntu arm64. No, troppo sbatty me la pesco a mando da aws

Poi faccio la spot fleet request. TODO foreach per ogni AZ
Poi gli do unip pubblico e un security group per fare almeno ssh. 