# HAProxy - Websockets

### Background

Quick article because it’s Friday afternoon and I’ll forget all about it next week.

Problem encountered was using HAProxy to handle websockets. This is used for the CUDOS Explorer, but am sure it’s not the last time we’ll need to do this.

My very basic understanding of websockets, ws and wss are equivalent to http and https.

Key difference for HAProxy is that http/https traffic is handled in http mode, ws and wss in tcp mode. Additionally, the ACL to match these two types differ.

### Key points

This works fine with Cloudflare proxy mode

Move `mode http` from the defaults section to be explicit in the front and backend for http/https

Set `mode tcp` to the front and back ends for websockets

Set the ACL for websockets to use y `ssl_fc_sni` this checks the SNI from the front connection `ssl_fc` - this was the bit that took longest to figure out

I wasn’t able to share port 80 to 443 redirect with both http and websocket traffic, the 80 to 443 redirect relies on `mode http`

### Config example

Here is the full config at the time of deploying:

```bash
global
    log /dev/log local0 debug

defaults
    log global
    timeout connect 10s
    timeout client 30s
    timeout server 30s

frontend port_80
    mode http
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

frontend port_443
    mode http
    bind *:443 ssl crt /etc/haproxy/certs/
    http-request add-header X-Forwarded-Proto https

    # ACL definitions for different hosts
    acl acl_for_grpc hdr_beg(host) -i grpc.cudos.org
    acl acl_for_rest hdr_beg(host) -i rest.cudos.org
    acl acl_for_rpc hdr_beg(host) -i rpc.cudos.org

    # Backend usage based on ACL
    use_backend cudos-noded-grpc if acl_for_grpc
    use_backend cudos-noded-rest if acl_for_rest
    use_backend cudos-noded-rpc if acl_for_rpc

frontend port_8443
    mode tcp
    option logasap
    option tcplog
    log global
    bind *:8443 ssl crt /etc/haproxy/certs/websocket.cudos.org_origin.pem

    # ACL definitions for different hosts
    acl acl_for_websocket ssl_fc_sni -i websocket.cudos.org

    # Backend usage based on ACL
    use_backend cudos-noded-websocket if acl_for_websocket

# Backend definitions
backend cudos-noded-grpc
    mode http
    server grpc-cudos-mainnet-fullnode-02 185.247.206.147:9090 check

backend cudos-noded-rest
    mode http
    server rest-cudos-mainnet-fullnode-02 185.247.206.147:1317 check
    http-response set-header Access-Control-Allow-Origin "*"
    http-response set-header Access-Control-Allow-Methods "GET, POST, OPTIONS"
    http-response set-header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept"
    http-response set-header Access-Control-Max-Age "3600"

backend cudos-noded-rpc
    mode http
    server rpc-cudos-mainnet-fullnode-02 185.247.206.147:26657 check
    http-response set-header Access-Control-Allow-Origin "*"
    http-response set-header Access-Control-Allow-Methods "GET, POST, OPTIONS"
    http-response set-header Access-Control-Allow-Headers "Origin, X-Requested-With, Content-Type, Accept"
    http-response set-header Access-Control-Max-Age "3600"

backend cudos-noded-websocket
    mode tcp
    timeout tunnel 1h
    server websocket-cudos-mainnet-fullnode-02 185.247.206.147:26657 check
```

### Troubleshooting

#### Option 1 - Pie Socket

For testing wss, Pie Socket allows web-based testing of wss connections [https://piehost.com/websocket-tester](https://piehost.com/websocket-tester)

For testing ws, Pie Socket has a Chrome browser extension to test unsecure websockets

#### Option 2 - wscat

Install wscat

```bash
apt update
apt install node-ws
```

Once installed, run the following

\#for wss\
wscat -c wss://websocket.cudos.org:8443/websocket

\#for ws\
wscat -c ws://185.247.206.147:26657/websocket
