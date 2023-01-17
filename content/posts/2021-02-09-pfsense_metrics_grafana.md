---
title: Monitoring pFsense with Cloud Graphite
date: 2021-02-09T21:21:12+01:00
tags:
    - networking

---

How to collect usage metrics from pfSense and send them to a Graphite-compatible backend

This post is specifically for [Grafana Cloud](https://grafana.com/products/cloud/), the m*metrics-as-a-service* platform that gives you a fully managed Grafana dashboard with a Graphite-compatible backend  

# 1. Install the Telegraf package for pfSense
Telegraf is an agent for collecting, processing, aggregating, and writing metrics. 

Telegraf is already packaged for pfSense. Simply  in the meu `System/Packages` and install it. 
{% include blog/figure.html img="pfsense-telegraf.png" alt="pfSense Telegraf package" caption="Install the Telegraf package for pfSense" %}


# 2. Install the carbon-relay-ng package
[carbon-relay-ng](https://github.com/grafana/carbon-relay-ng) is a relay for carbon streams, in go. Like carbon-relay from the graphite project, except it supports Grafana Cloud authentication (and many other things....)

Unfortunately, there is no pre-built bianry for the FreeBSD/386 architecture. You can either pull the source code and build it, or use my UNOFFICIAL and USNUPPORTED build [here](https://github.com/totomz/carbon-relay-ng/releases)  

# 3. Configure carbon-relay and Telegraf plugin

## Telegraf plugin
The configuration is straight-forward, just choose `Graphite` as the preferred output, and point Telegraf to the local carbon-relay-ng, by default it will be `localhost:2003`

{% include blog/figure.html img="telegraf-config.png" alt="pfSense Telegraf package configuration" caption="Configure the Telegraf package to use Graphite on localhost:2003" %}

# carbon-realy-ng

The configuration of carbon-relay-ng is straight forward, simply copy-pase the examples on your Grafana Cloud dashboards or uset the following snippets:

### /etc/carbon-relay-ng/carbon-relay-ng.conf
```
## Global settings ##
# instance id's distinguish stats of multiple relays.
# do not run multiple relays with the same instance id.
# supported variables:
#  ${HOST} : hostname
instance = "${HOST}"

## System ##
# this setting can be used to override the default GOMAXPROCS logic
# it is ignored if the GOMAXPROCS environment variable is set
max_procs = 2
pid_file = "carbon-relay-ng.pid"
# directory for spool files
spool_dir = "spool"

## Logging ##
# one of trace debug info warn error fatal panic
# see docs/logging.md for level descriptions
# note: if you used to use "notice", you should now use "info".
log_level = "info"

## Inputs ##
### plaintext Carbon ###
listen_addr = "0.0.0.0:2003"
# close inbound plaintext connections if they've been idle for this long ("0s" to disable)
plain_read_timeout = "2m"
### Pickle Carbon ###
pickle_addr = "0.0.0.0:2013"
# close inbound pickle connections if they've been idle for this long ("0s" to disable)
pickle_read_timeout = "2m"

## Validation of inputs ##
# you can also validate that each series has increasing timestamps
validate_order = false

# How long to keep track of invalid metrics seen
# Useful time units are "s", "m", "h"
bad_metrics_max_age = "24h"

[[route]]
key = 'grafanaNet'
type = 'grafanaNet'
addr = 'https://graphite-us-central1.grafana.net/metrics'
apikey = '28645:<Your Grafana.com API Key>'
schemasFile = '/etc/carbon-relay-ng/storage-schemas.conf'

## Instrumentation ##
[instrumentation]
# in addition to serving internal metrics via expvar, you can send them to graphite/carbon
# IMPORTANT: setting this to "" will disable flushing, and metrics will pile up and lead to OOM
# see https://github.com/graphite-ng/carbon-relay-ng/issues/50
# so for now you MUST send them somewhere. sorry.
# (Also, the interval here must correspond to your setting in storage-schemas.conf if you use Grafana Cloud)
graphite_addr = "localhost:2003"
graphite_interval = 10000  # in ms
```

### /etc/carbon-relay-ng/storage-schemas.conf 
```
[default]
  pattern = .*
  retentions = 10s:1d
```

# Configure rc
This is a baseline configuration for the rc init system. Save it in `/etc/rc.d/carbon-relay-ng`:
```
#!/bin/sh

. /etc/rc.subr

# General Info
name="carbonrelayng"            # Safe name of program
program_name="carbon-relay-ng"   # Name of exec
title="carbon-relay-ng"          # Title to display in top/htop

# RC.config vars
load_rc_config $name


pidfile="/var/run/${program_name}.pid"

exec_path="/root/carbon-relay-ng"
output_file="/var/log/${program_name}.log"

command="/usr/sbin/daemon"
command_args="-r -t ${title} -u root -o ${output_file} -P ${pidfile} ${exec_path} "

# Loading Config
load_rc_config ${name}
run_rc_command "$1"
```

Now, carbon-relay-ng can be easily managed with `/etc/init.d/carbon-relay-ng start|stop|enable`