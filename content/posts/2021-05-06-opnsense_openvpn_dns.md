---
title: OpenVPN + OPNSense - fix DNS
date: 2021-05-06T21:21:12+01:00
tags: 
    - opnsense
    - networking

---

> This it a TL;DR post to fix a DNS issue with OpenVPN and OPNSense. 


# The vpn client connects but dns resolution is not working
**Symptoms**: 
* the client connects to the VPN server;
* you can reach a remote server by ip: `ping 8.8.8.8` succeed
* DNS queries are failing:

```
(base) ➜  ~ dig www.google.com

; <<>> DiG 9.10.6 <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 55522
;; flags: qr rd ad; QUERY: 0, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; Query time: 62 msec
;; SERVER: 192.168.10.1#53(192.168.10.1)
;; WHEN: Thu May 06 11:35:48 CEST 2021
;; MSG SIZE  rcvd: 12
```

**Solution:**
Check the `dig` response, in the `->>HEADER<<-` section you should be able to see a `status: REFUSED` header.

If this is the case, you need to explicitly add the OpenVPN subnet to the ACL of UnboundDNS:

1) Go to your OPNSense admin page
2) Go to `Services -> Unbound DNS -> Access List`
3) Add a new rule, enabling **the whole vpn subnet**


By default, OPNSense whitelist only the OpenVPN server, not the entire vpn tunnel.