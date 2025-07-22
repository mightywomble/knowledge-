# Install a Local version of Netdata

## Introduction

Netdata is a great product, I’ve been using it for years. In the past year or so, to pay the bills, they have really pushed the [netdata.cloud](http://netdata.cloud) functionality. If you are self hosting however Cudos do not want their monitoring data to end up in the cloud.

## Outcome

The purpose of this post

* install the offline version of Netdata,
* lock it down to specific IP’s for Access
* create a parent node which all other servers on your infrastructure can push data to
* provide a single pane of glass
* Do all of this using Ansible.
* Have no interface with netdata.cloud

## Repository:

The Ansible to do this can be found here:

[https://github.com/mightywomble/netdata\_install](https://github.com/mightywomble/netdata_install)

## Setup

### Create the Parent Server

The first place to start is with the Parent server, this is the server which you will access the dashboard and point all the child servers data to.

| **Item** | **Spec**          |
| -------- | ----------------- |
| RAM      | 4Gb               |
| DISK     | 100Gb             |
| CPU      | 2vCPU             |
| OS       | Linux (Debian 12) |

### Install the Parent Server

**Netdata Offline Install:** [https://learn.netdata.cloud/docs/netdata-agent/installation/linux/offline-systems](https://learn.netdata.cloud/docs/netdata-agent/installation/linux/offline-systems)

Use the netdata offline install method:

```bash
curl <https://get.netdata.cloud/kickstart.sh> > /tmp/netdata-kickstart.sh && sh /tmp/netdata-kickstart.sh --prepare-offline-install-source ./netdata-offline
```

```bash
cd netdata-offline
```

```bash
./install.sh
```

This will create a working netdata server on `http://<server ip>:19999`

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/ee471fa0-6f56-4e97-8e26-3ebad77254b7/image.png)

### Configure Parent server

This server is, if your IP is) internet accessible. I run my server on a public VPS, to limit who can access the web interface, we make changes in `netdata.conf`

**Netdata Registry:** [https://learn.netdata.cloud/docs/netdata-agent/registry](https://learn.netdata.cloud/docs/netdata-agent/registry)

### → Edit netdata.conf

On the netdata server head to

```bash
cd /opt/netdata/etc/netdata 
```

Edit netdata.conf

```bash
sudo nano netdata.conf
```

Make the following changes (remove # the symbol)

```bash
**[logs]**
# debug flags = 0x0000000000000000
**facility = daemon**
# logs flood protection period = 60
# logs to trigger flood protection = 1000
# level = info
# debug = /opt/netdata/var/log/netdata/debug.log
**daemon = journal
collector = journal**
# access = /opt/netdata/var/log/netdata/access.log
**health = journal**
```

```bash
**[ml]**
enabled = yes
# maximum num samples to train = 21600
# minimum num samples to train = 900
# train every = 10800
# number of models per dimension = 18
# delete models older than = 604800
# num samples to diff = 1
# num samples to smooth = 3
# num samples to lag = 5
# random sampling ratio = 0.20000
# maximum number of k-means iterations = 1000
# dimension anomaly score threshold = 0.99000
# host anomaly rate threshold = 1.00000
# anomaly detection grouping method = average
# anomaly detection grouping duration = 300
# num training threads = 4
# flush models batch size = 128
# dimension anomaly rate suppression window = 900
# dimension anomaly rate suppression threshold = 450
# enable statistics charts = no
# hosts to skip from training = !*
# charts to skip from training = netdata.*
# stream anomaly detection charts = yes
```

```bash
**[health]**
# silencers file = /opt/netdata/var/lib/netdata/health.silencers.json
**enabled = yes
enable stock health configuration = yes
use summary for notifications = yes**
# default repeat warning = never
# default repeat critical = never
# in memory max health log entries = 1000
**health log history = 432000
script to execute on alarm = /opt/netdata/usr/libexec/netdata/plugins.d/alarm-notify.sh
enabled alarms = ***
# run at least every seconds = 10
# postpone alarms during hibernation for seconds = 60
```

```bash
**[web]**
# ssl key = /opt/netdata/etc/netdata/ssl/key.pem
# ssl certificate = /opt/netdata/etc/netdata/ssl/cert.pem
# tls version = 1.3
# tls ciphers = none
# ses max tg_des_window = 15
# des max tg_des_window = 15
# mode = static-threaded
# listen backlog = 4096
# default port = 19999
# bind to = *
# bearer token protection = no
# disconnect idle clients after seconds = 60
# timeout for first request = 60
# accept a streaming request every seconds = 0
# respect do not track policy = no
# x-frame-options response header =
**allow connections from = 88.98.100.166 100.***
# allow connections by dns = heuristic
# allow dashboard from = localhost *
# allow dashboard by dns = heuristic
# allow badges from = *
# allow badges by dns = heuristic
**allow streaming from = 100.***
# allow streaming by dns = heuristic
# allow netdata.conf from = localhost fd* 10.* 192.168.* 172.16.* 172.17.* 172.18.* 172.19.* 172.20.* 172.21.* 172.22.* 172.23.* 1>
# allow netdata.conf by dns = no
# allow management from = localhost
# allow management by dns = heuristic
# enable gzip compression = yes
# gzip compression strategy = default
# gzip compression level = 3
# ssl skip certificate verification = no
# web server threads = 4
# web server max sockets = 131072
```

on the two edited lines the IP’s you allow access to the web interface from are either the full or wildcard addresses

```bash
**[registry]
enabled = yes**
# netdata unique id file = /opt/netdata/var/lib/netdata/registry/netdata.public.unique.id
# registry db file = /opt/netdata/var/lib/netdata/registry/registry.db
# registry log file = /opt/netdata/var/lib/netdata/registry/registry-log.db
# registry save db every new entries = 1000000
# registry expire idle persons days = 365
# registry domain =
# registry to announce = <https://registry.my-netdata.io>
# registry hostname = netdata
# verify browser cookies support = yes
# enable cookies SameSite and Secure = yes
# max URL length = 1024
# max URL name length = 50
# use mmap = no
# netdata management api key file = /opt/netdata/var/lib/netdata/netdata.api.key
**allow from = 88.98.100.166 100.***
# allow by dns = heuristic
```

Save the file

Restart the Netdata Service

```bash
sudo systemctl restart netdata
```

Test access is restricted to the Web Interface to the IP(s) you specified.

### Setup for Streaming:

With the Parent server setup and locked down, it needs to be setup for streaming. this is allowing the child nodes to send data to the parent node.

**Setup streaming:** [https://learn.netdata.cloud/docs/observability-centralization-points/metrics-centralization-points/configuring-metrics-centralization-points](https://learn.netdata.cloud/docs/observability-centralization-points/metrics-centralization-points/configuring-metrics-centralization-points)

#### → Create an API token

The Parent is going to be locked to only allowing access to child nodes with the same API string enabled.

```bash
uuidgen
```

This will generate a string

```bash
4b198662-41cc-4eb9-972b-64e01a78df01
```

Copy this, you’ll need it in a minute

#### → Edit Streaming config

From the same folder

```bash
cd /opt/netdata/etc/netdata 
```

This time run

```bash
sudo ./edit-config stream.conf
```

Find the section marked

\[API]

and make the following changes

```bash
**[9dc75f24-eeab-41cd-9ca6-55c0a4ac55bf]**
# Default settings for this API key

# This GUID is to be used as an API key from remote agents connecting
# to this machine. Failure to match such a key, denies access.
# YOU MUST SET THIS FIELD ON ALL API KEYS.
type = api

# You can disable the API key, by setting this to: no
# The default (for unknown API keys) is: no
enabled = yes

# A list of simple patterns matching the IPs of the servers that
# will be pushing metrics using this API key.
# The metrics are received via the API port, so the same IPs
# should also be matched at netdata.conf [web].allow connections from
allow from = 100.*

# Shall we enable health monitoring for the hosts using this API key?
# 3 possible values:
#    yes     enable alarms
#    no      do not enable alarms
#    auto    enable alarms, only when the sending netdata is connected.
#            Health monitoring will be disabled as soon as the connection is closed.
# You can also set it per host, below.
# The default is taken from [health].enabled of netdata.conf
health enabled by default = yes
```

Change

**\[api]** to **\[4b198662-41cc-4eb9-972b-64e01a78df01]**

In this example, I have my nodes on a subnet of 100.100.10.0/24 so I’ve set

```bash
allow from = 100.*
```

I could allow over the public interface as well

```bash
**allow from = 88.98.100.166 100.***
```

Save and exit the file

Restart the Netdata Service

```bash
sudo systemctl restart netdata
```

The parent server is now setup



## Netdata Client Server Setup

The document on this page outlines how every server which you wish to show up in the Master server will need to be setup. there is supplied ansible code available to automate this, as well as understanding the manual settings.

## Install

The Installation is the same as the Parent server

**Netdata Offline Install:** [https://learn.netdata.cloud/docs/netdata-agent/installation/linux/offline-systems](https://learn.netdata.cloud/docs/netdata-agent/installation/linux/offline-systems)

Use the netdata offline install method:

```bash
curl <https://get.netdata.cloud/kickstart.sh> > /tmp/netdata-kickstart.sh && sh /tmp/netdata-kickstart.sh --prepare-offline-install-source ./netdata-offline
```

```bash
cd netdata-offline
```

```bash
./install.sh
```

This will create a working netdata server on `http://<server ip>:19999`

### Web Interface

If you have any servers you’ll be monitoring with Netdata which are public to the internet, you may want to lock them down to access only from your public IP

On the netdata child server head to

```bash
cd /opt/netdata/etc/netdata 
```

Edit netdata.conf

```bash
sudo nano netdata.conf
```

Make the following changes (remove # the symbol)

```bash
**[web]**
# ssl key = /opt/netdata/etc/netdata/ssl/key.pem
# ssl certificate = /opt/netdata/etc/netdata/ssl/cert.pem
# tls version = 1.3
# tls ciphers = none
# ses max tg_des_window = 15
# des max tg_des_window = 15
# mode = static-threaded
# listen backlog = 4096
# default port = 19999
# bind to = *
# bearer token protection = no
# disconnect idle clients after seconds = 60
# timeout for first request = 60
# accept a streaming request every seconds = 0
# respect do not track policy = no
# x-frame-options response header =
**allow connections from = 88.98.100.166 100.***
# allow connections by dns = heuristic
# allow dashboard from = localhost *
# allow dashboard by dns = heuristic
# allow badges from = *
# allow badges by dns = heuristic
**allow streaming from = 100.***
# allow streaming by dns = heuristic
# allow netdata.conf from = localhost fd* 10.* 192.168.* 172.16.* 172.17.* 172.18.* 172.19.* 172.20.* 172.21.* 172.22.* 172.23.* 1>
# allow netdata.conf by dns = no
# allow management from = localhost
# allow management by dns = heuristic
# enable gzip compression = yes
# gzip compression strategy = default
# gzip compression level = 3
# ssl skip certificate verification = no
# web server threads = 4
# web server max sockets = 131072
```

These are the IP Adderesses which will be able to access the Web interface, in this example its an external IP as the server has external internet access, this could be an internal NAT range as well

### Streaming

From the same folder

This time run

```bash
sudo nano /opt/netdata/usr/lib/netdata/conf.d/stream.conf
```

Find the section marked

**\[stream]**

change the following settings

```bash
enabled = yes
destination = http://<ip/dns parent server>:19999
api key = <the Api key used above on the server>
```

Save and exit the file

Restart the Netdata Service

```bash
sudo systemctl restart netdata
```

The child server is now setup

### Test

#### Child Dashboard

Open the url for the Child server http://\<child address>:19999

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/ee471fa0-6f56-4e97-8e26-3ebad77254b7/image.png)

#### Parent Dashboard

Open the url for the Parent server http://\<child address>:19999

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/30f1219c-b8c4-4b87-902e-e1eefe28272e/image.png)

Click On Nodes

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/4deac958-18e1-4ac7-8faf-cbf746a356b3/image.png)

All your Child nodes are listed

Clicking on a child node will open its dashboard (on the parent server)

## Ansible

I’ve created some ansible which using vars/main.yaml will setup basic child streaming nodes here:

[https://github.com/mightywomble/netdata\_install](https://github.com/mightywomble/netdata_install)

## → Alerting / Notifications

**Agent Notifications:** [https://learn.netdata.cloud/docs/alerts-&-notifications/notifications/agent-dispatched-notifications](https://learn.netdata.cloud/docs/alerts-&-notifications/notifications/agent-dispatched-notifications)

With this setup we have set the child health notifications to forward via the streaming interface to the parent server. Doing this means we can setup the parent server to forward notifications to systems like email,discord or slack (for example)

#### → to Discord

**Discord Notifications**: [https://learn.netdata.cloud/docs/alerts-&-notifications/notifications/agent-dispatched-notifications/discord](https://learn.netdata.cloud/docs/alerts-&-notifications/notifications/agent-dispatched-notifications/discord)

**From: Parent server**

Head to the netdata config folder

```bash
cd /opt/netdata/etc/netdata
```

run

```bash
sudo ./edit-config health_alarm_notify.conf
```

Find the Discord section

```bash
#------------------------------------------------------------------------------
# discord (discord.com) global notification options

# multiple recipients can be given like this:
#                  "CHANNEL1 CHANNEL2 ..."

# enable/disable sending discord notifications
SEND_DISCORD="YES"

# Create a webhook by following the official documentation -
# <https://support.discord.com/hc/en-us/articles/228383668-Intro-to-Webhooks>
DISCORD_WEBHOOK_URL="<https://discord.com/api/webhooks/127733463456346956293/hYcaEdRvCiuNzG531FxKfLjXP-HN5N2zMCsdggsfsdfggsdFpdCd-UKCtCkCm8viG_u>"

# if a role's recipients are not configured, a notification will be send to
# this discord channel (empty = do not send a notification for unconfigured
# roles):
DEFAULT_RECIPIENT_DISCORD="netdata_alerts"
```

Add your Webhook URL and other changes

Save and Exit

Restart the Netdata Server

```bash
sudo systemctl restart netdata
```

#### → Notes:

*

````
| **Name** | **Description** | **Default** | **Required** |
| --- | --- | --- | --- |
| SEND_DISCORD | Set **`SEND_DISCORD`** to YES | YES | yes |
| DISCORD_WEBHOOK_URL | set **`DISCORD_WEBHOOK_URL`** to your webhook URL. |  | yes |
| DEFAULT_RECIPIENT_DISCORD | Set **`DEFAULT_RECIPIENT_DISCORD`** to the channel you want the alert notifications to be sent to. You can define multiple channels like this: **`alerts`** **`systems`**. |  | yes |

### DEFAULT_RECIPIENT_DISCORD

All roles will default to this variable if left unconfigured. You can then have different channels per role, by editing **`DEFAULT_RECIPIENT_DISCORD`** with the channel you want, in the following entries at the bottom of the same file:

```
role_recipients_discord[sysadmin]="systems"
role_recipients_discord[domainadmin]="domains"
role_recipients_discord[dba]="databases systems"
role_recipients_discord[webmaster]="marketing development"
role_recipients_discord[proxyadmin]="proxy-admin"
role_recipients_discord[sitemgr]="sites"

```

The values you provide should already exist as Discord channels in your server.
````

#### **Troubleshooting**

#### **→ Test Notification**

You can run the following command by hand, to test alerts configuration:

```bash
# become user netdata
sudo su -s /bin/bash netdata

# enable debugging info on the console
exportNETDATA_ALARM_NOTIFY_DEBUG=1

# send test alarms to sysadmin
/usr/libexec/netdata/plugins.d/alarm-notify.sh test

# send test alarms to any role
/usr/libexec/netdata/plugins.d/alarm-notify.sh test "ROLE"

```

Note that this will test _all_ alert mechanisms for the selected role.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/b5279a39-9358-4b33-8e67-9bdfe4055c73/image.png)
