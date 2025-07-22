# HAProxy Install

## Manual Install Commands

Install haproxy

```jsx
sudo apt install haproxy
```

## Manual Post Install Setup

the server is going to use a multi configuration setup, to allow any service to be added in a modular approach rather than a monolithic haproxy.cfg, this is easier to manager and update.

To do this a conf.d folder is setup

### Directories

```jsx
sudo mkdir -p /etc/haproxy/conf.d
```

### systemd setup

By default haproxy only looks at the file `/etc/haproxy/haproxy.cfg` when it starts up. we need to change this.

```jsx
sudo nano /lib/systemd/system/haproxy.service
```

change to

```jsx
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target rsyslog.service
Wants=network-online.target

[Service]
EnvironmentFile=-/etc/default/haproxy
EnvironmentFile=-/etc/sysconfig/haproxy
BindReadOnlyPaths=/dev/log:/var/lib/haproxy/dev/log
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "CONFIGDIR=/etc/haproxy/conf.d/" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
KillMode=mixed
Restart=always
SuccessExitStatus=143
Type=notify

[Install]
WantedBy=multi-user.target

```

These are the lines which contain the cariables which will indiate tot he service to also use the /etc/haproxy/conf.d folder

```jsx
Environment="CONFIG=/etc/haproxy/haproxy.cfg" "CONFIGDIR=/etc/haproxy/conf.d/" "PIDFILE=/run/haproxy.pid" "EXTRAOPTS=-S /run/haproxy-master.sock"
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -p $PIDFILE $EXTRAOPTS
ExecReload=/usr/sbin/haproxy -Ws -f $CONFIG -f $CONFIGDIR -c -q $EXTRAOPTS
ExecReload=/bin/kill -USR2 $MAINPID
```

Reload the service

```jsx
sudo systemctl daemon-reload
```

Restart the service

```jsx
sudo systemctl restart haproxy
```

**Note: this may fail as we have no configs available**

## haproxy.cfg

This is the base HAProxy configuration

`/etc/haproxy/haproxy.cfg`

```jsx
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 50000

    # Default ciphers to use on SSL-enabled listening sockets
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
    ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

    tune.ssl.default-dh-param 2048

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 10s
    timeout client  1m
    timeout server  1m
    option http-keep-alive
    option http-server-close
    compression algo gzip
    compression type text/html text/plain text/css application/javascript
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:bky9DHZ@pjk_aqe4efd

cache static-cache
    total-max-size 256    # Cache size in megabytes
    max-age 240           # Maximum age of cached objects in seconds

```

Some things to note here

#### Logging

Logging is pushed to journald using thesee config items

```jsx
log /dev/log local0
log /dev/log local1 notice
daemon
```

#### Config WebUI

This config will presnt a local webUI for HAProxy

```jsx
frontend stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats auth admin:bky9DHZ@pjk_aqe4efd
```

#### Static Cache

This config helps reduce the latency of the proxy

```jsx
cache static-cache
    total-max-size 256    # Cache size in megabytes
    max-age 240           # Maximum age of cached objects in seconds
```

## frontend config

`/etc/haproxy/conf.d/00-frontend.cfg`

The frontend config is handled by a single file, this file will accept traffic in on port 443/tcp (https) and assess it aagainst an Access Control List

```jsx
frontend https_frontend
    bind *:443 ssl crt /opt/cloudflare/certs/wildcard/cert.pem alpn h2,http/1.1

    # Use the real client IP from Cloudflare
    http-request set-src req.hdr(CF-Connecting-IP)

    # ACLs to match hostnames
    acl host_netdata hdr(host) -i netdata.cudos.org
    acl host_graylog hdr(host) -i graylog.cudos.org

    # Check if the request is coming from Cloudflare
    acl cloudflare_ips src -f /etc/haproxy/cloudflare_ips.lst

    # Allow if from Cloudflare, deny otherwise
    http-request deny unless cloudflare_ips

    # Use backends based on hostname
    use_backend netdata_backend if host_netdata
    use_backend graylog_backend if host_graylog

    # Default backend if no match
    default_backend default_backend
```

#### Dataflow

[https://netdata.cudos.org](https://netdata.cudos.org) → check cert → haproxy front end

ACL Check if listed

If not listed redirect to [https://cudos.org](https://cudos.org)

If is Listed Check if request is coming from a cloudflare or an approved IP (see IP Access below)

If passed use backend netdata\_backend.

## backend config

In the example there are 2 backend server configs

`/etc/haproxy/conf.d/01-graylog.cfg`

```jsx
backend graylog_backend
    mode http
    balance roundrobin
    option forwardfor
    http-reuse safe
    server graylog_server 100.67.124.62:9000 check inter 5s fall 3 rise 2
    timeout connect 10s
    timeout server 30s
    retries 3

```

`/etc/haproxy/conf.d/02-netdata.cfg`

```jsx
backend netdata_backend
    mode http
    balance roundrobin
    option forwardfor
    http-reuse safe
    server netdata_server 100.67.253.39:19999 check inter 5s fall 3 rise 2
    timeout connect 10s
    timeout server 30s
    retries 3
```

Both configurations are pretty similar and do the same thing

forward traffic to a netbird internal IP Address

## IP Access list

`/etc/haproxy/cloudflare_ips.lst`

the proxy is locked down to only provide access to people coming in via a cloudflarte proxy IP Address. However there are two checks here

```jsx
88.98.84.166/32  <- My Home IP
103.21.244.0/22
103.22.200.0/22
103.31.4.0/22
104.16.0.0/13
104.24.0.0/14
108.162.192.0/18
131.0.72.0/22
141.101.64.0/18
162.158.0.0/15
172.64.0.0/13
173.245.48.0/20
188.114.96.0/20
190.93.240.0/20
197.234.240.0/22
198.41.128.0/17

```

Because I’ve applied the cloudflare proxy to both urls [netdata.cudos.org](http://netdata.cudos.org) and [graylog.cudos.org](http://graylog.cudos.org) the service will first check the traffic is coming in via a cloudflare proxy ip, then will look to my external IP to allow direct access to a service.

Note: Even with all this working, if graylog has UFW setup it may not allow acces (it does) or netdata is setup in its netdata.conf to only allow access to specific IP addresses (it is) then this too may not allow Access.

## Cloudflare certificates

I’ve utilised a Cloudflare origin certificate and the files are stored on the server as

`/opt/cloudflare/certs/wildcard/cert.pem`

`/opt/cloudflare/certs/wildcard/cert.pem.key`







