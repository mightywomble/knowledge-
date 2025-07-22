# Centralising Debian12 Journald logs using Filebeat and Graylog

I’ve recently been looking into centralizing logs from several servers and being able to search them and things have changed a bit. `syslog` is no longer the default for the distros I use, `journald` is the new kid on the block, and this brings a few new challenges.

This post goes over how i setup Graylog (6.x at time of writing) on Debian12, and used Filebeats on debian12 clients to push `journalctl` logs into Graylog.

This wasn’t without its issues, mainly because the Graylog install docs for Debian are terrible, no they actually are.. I’ll indicate where below.

## Server Setup

Some notes and observations

### → Ram/CPU

Don’t be scared here, you’re going to need as much as you can spare, especially on a busy servers. I’ve used 8vCPU’s and 12Gb Ram on a Proxmox VPS

### → Proxmox

Its important when you setup the virtual machine, the CPU type is set to `host`

If you don’t do this, Mongo will not start

**REFERENCE**: [https://stackoverflow.com/questions/76126384/mongodb-5-0-requires-a-cpu-with-avx-support-container-failed-to-start](https://stackoverflow.com/questions/76126384/mongodb-5-0-requires-a-cpu-with-avx-support-container-failed-to-start)

## Server Installation - Graylog

### → Mongo Db

**REFERENCE**: [https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-debian/](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-debian/)

The Instructions on the Graylog page refer to version 6.x of Mongodb which just do not work, they do refer to a link to the 7.x install which is the one to follow.

```jsx
sudo apt-get install gnupg curl
```

Import the key

```jsx
curl -fsSL <https://www.mongodb.org/static/pgp/server-7.0.asc> | \\   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \\
   --dearmor
```

Add the repo

```jsx
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] <http://repo.mongodb.org/apt/debian> bookworm/mongodb-org/7.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```

Update apt

```jsx
sudo apt-get update
```

Install MongoDb

```jsx
sudo apt-get install -y mongodb-org
```

Note:

If the `ulimit` value for number of open files is under `64000`, MongoDB generates a startup warning.

```jsx
ulimit -n 64000
```

Start the MongoD service

```jsx
sudo systemctl daemon-reload
sudo systemctl enable mongod.servicesudo systemctl restart mongod.servicesudo systemctl --type=service --state=active | grep mongod
```

check its running

```jsx
sudo systemctl status mongod.service
```

should show

```jsx
● mongod.service - MongoDB Database Server     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-04 10:23:18 UTC; 9s ago
       Docs: <https://docs.mongodb.org/manual>   Main PID: 2338 (mongod)
     Memory: 76.0M
        CPU: 639ms     CGroup: /system.slice/mongod.service             └─2338 /usr/bin/mongod --config /etc/mongod.conf
```

Lets mark mongo as hold so apt doesn’t upgrade it

```jsx
sudo apt-mark hold mongodb-org
```

### → Opensearch

#### Install Opensearch

Openseach is providing the eleastic type search capabilities for Graylog, I’m running this as an All in One server, you could run OpenSearch on a seperate server if you wanted to.

Install needed files

```jsx
sudo apt-get update && sudo apt-get -y install lsb-release ca-certificates curl gnupg2
```

Install the repo gpg key

```jsx
curl -o- <https://artifacts.opensearch.org/publickeys/opensearch.pgp> | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
```

Add the repo list

```jsx
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] <https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt> stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```

Update apt

```jsx
sudo apt-get update
```

**Note:**

To move forward we need to generate an admin key for Opensearch

Generate a Key

```jsx
 tr -dc A-Z-a-z-0-9_@#%^-_=+ < /dev/urandom  | head -c${1:-32}
```

This will return a random key like:

```jsx
2EhY78l0_cY+6o6YJzD8GUnodOJGO0K6
```

Now Install Opensearch using the key

```jsx
sudo OPENSEARCH_INITIAL_ADMIN_PASSWORD=2EhY78l0_cY+6o6YJzD8GUnodOJGO0K6 apt-get -y install opensearch
```

Hold the version of Openseach (upgrading it might cause problems with an apt update)

```jsx
sudo apt-mark hold opensearch
```

We won’t start the service yet as there are config changes to make

#### Opensearch Graylog config

Edit the Openseach config

```jsx
sudo nano /etc/opensearch/opensearch.yml
```

Edit/Add the following

```jsx
# ---------------------------------- Cluster -----------------------------------#
# Use a descriptive name for your cluster:#
cluster.name: homelan
#
# ------------------------------------ Node ------------------------------------#
# Use a descriptive name for the node:#
node.name: ${HOSTNAME}
#
# Add custom attributes to the node:#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------#
# Path to directory where to store the data (separate multiple locations by comma):#
path.data: /var/lib/opensearch## Path to log files:#path.logs: /var/log/opensearch#
# ----------------------------------- Added Info -----------------------------------discovery.type: single-node
network.host: 0.0.0.0action.auto_create_index: falseplugins.security.disabled: true
```

save and exit

Edit the Java Options

```jsx
sudo nano /etc/opensearch/jvm.options
```

Change the settings

```jsx
-Xms1g
-Xmx1g
```

to 50% of your servers ram

```jsx
-Xms4g
-Xmx4g
```

Save and exit

Edit sysconfig

```jsx
sudo nano /etc/sysctl.conf
```

add the line to the end of the file

```jsx
vm.max_map_count=262144
```

This will apply the setting over a reboot, to apply it now use

```jsx
sudo sysctl -w vm.max_map_count=262144
```

#### Start Opensearch

Start the Opensearch service

```jsx
sudo systemctl daemon-reload
sudo systemctl enable opensearch.servicesudo systemctl start opensearch.service
```

Check its running

```jsx
sudo systemctl status opensearch.service
```

will return

```jsx
● opensearch.service - OpenSearch
     Loaded: loaded (/lib/systemd/system/opensearch.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-04 11:52:40 UTC; 1min 6s ago
       Docs: <https://opensearch.org/>   Main PID: 3763 (java)
      Tasks: 79 (limit: 9358)
     Memory: 4.3G
        CPU: 33.293s
     CGroup: /system.slice/opensearch.service             └─3763 /usr/share/opensearch/jdk/bin/java -Xshare:auto -Dopensearch.networkaddress.cache.ttl=60 -Dopensearch.networkaddress.cache.negative.ttl=>Sep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager has been called by org.opensearch.bootstrap.OpenSearch (file:/usr/shar>Sep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: Please consider reporting this to the maintainers of org.opensearch.bootstrap.OpenSearchSep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager will be removed in a future release
Sep 04 11:52:31 logging systemd-entrypoint[3763]: Sep 04, 2024 12:52:31 PM sun.util.locale.provider.LocaleProviderAdapter <clinit>Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: COMPAT locale provider will be removed in a future release
Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: A terminally deprecated method in java.lang.System has been called
Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager has been called by org.opensearch.bootstrap.Security (file:/usr/share/>Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: Please consider reporting this to the maintainers of org.opensearch.bootstrap.SecuritySep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager will be removed in a future release
Sep 04 11:52:40 logging systemd[1]: Started opensearch.service - OpenSearch.
```

I’ve ignored the errors for a started service

### → Graylog

#### Install Server

Lets Install the graylog server

```jsx
wget <https://packages.graylog2.org/repo/packages/graylog-6.0-repository_latest.debsudo> dpkg -i graylog-6.0-repository_latest.debsudo apt-get update
sudo apt-get install graylog-server
```

Hold the version in apt

```jsx
sudo apt-mark hold graylog-server
```

#### Configure Server

We need 2 passwords for the config file

`password_secret` and `root_password_sha2`

to generate `password_secret` run

```jsx
< /dev/urandom tr -dc A-Z-a-z-0-9 | head -c${1:-96};echo;
```

This will generate a long string copy this, you’ll need it in the config file

```jsx
CjXGDLNLZskEJdPPKnEd1UnvJfsU7iAnePevgw-0Sq0K7ynw1sC3WDF-8L09SHG6lZqCGFweTAhLU1ai2zLFszc9LvbM0CWP
```

to generate the `root_password_sha2` run:

```jsx
echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\\n' | sha256sum | cut -d" " -f1
```

This will prompt you to enter a password.

This is the password for the admin user on the Graylog web interface

It will generate a long string

```jsx
5fab5f43d0b2f97526e7daacb1e42ffd178e2067c59a0036d0d984a3bec289fb
```

again. save this.

Edit the graylog server config

```jsx
sudo nano /etc/graylog/server/server.conf
```

Find the two password sections and add the generated strings

```jsx
# You MUST set a secret to secure/pepper the stored user passwords here. Use at least 64 characters.# Generate one by using for example: pwgen -N 1 -s 96# ATTENTION: This value must be the same on all Graylog nodes in the cluster.# Changing this value after installation will render all user sessions and encrypted values in the database invalid. (e.g. encrypted access tokens)
password_secret = CjXGDLNLZskEJdPPKnEd1UnvJfsU7iAnePevgw-0Sq0K7ynw1sC3WDF-8L09SHG6lZqCGFweTAhLU1ai2zLFszc9LvbM0CWP# The default root user is named 'admin'#root_username = admin
# You MUST specify a hash password for the root user (which you only need to initially set up the
# system and in case you lose connectivity to your authentication backend)
# This password cannot be changed using the API or via the web interface. If you need to change it,# modify it in this file.# Create one by using for example: echo -n yourpassword | shasum -a 256# and put the resulting hash value into the following line
**root_password_sha2 = 5fab5f43d0b2f97526e7daacb1e42ffd178e2067c59a0036d0d984a3bec289fb**
```

Find `http_bind_address`

and change it to this

```jsx
http_bind_address = 0.0.0.0:9000
```

Find `elasticsearch_hosts`

and change it to this

```jsx
elasticsearch_hosts = <http://127.0.0.1:9200>
```

Save and exit the file

#### Start Server

Run the following

```jsx
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.servicesudo systemctl start graylog-server.servicesudo systemctl --type=service --state=active | grep graylog
```

Check Everything is running

```jsx
sudo systemctl status graylog-server
```

Should return this

```jsx
● graylog-server.service - Graylog server
     Loaded: loaded (/lib/systemd/system/graylog-server.service; enabled; preset: enabled)
     Active: active (running) since Wed 2024-09-04 12:09:22 UTC; 21s ago
       Docs: <http://docs.graylog.org/>   Main PID: 5236 (graylog-server)
      Tasks: 120 (limit: 9358)
     Memory: 809.0M
        CPU: 33.887s
     CGroup: /system.slice/graylog-server.service             ├─5236 /bin/sh /usr/share/graylog-server/bin/graylog-server
             └─5237 /usr/share/graylog-server/jvm/bin/java -Xms1g -Xmx1g -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow -Djdk.tls.acknowledgeCloseNotif>Sep 04 12:09:22 logging systemd[1]: Started graylog-server.service - Graylog server.
```

### → Check all services running

At this point we can run some checks to make sure everything is still running

#### Mongodb

```jsx
sudo systemctl status mongod
```

```jsx
● mongod.service - MongoDB Database Server     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; preset: enabled)
     **Active: active (running) since Wed 2024-09-04 10:23:18 UTC; 1h 49min ago**       Docs: <https://docs.mongodb.org/manual>   Main PID: 2338 (mongod)
     Memory: 143.6M
        CPU: 31.374s
     CGroup: /system.slice/mongod.service             └─2338 /usr/bin/mongod --config /etc/mongod.confSep 04 10:23:18 logging systemd[1]: Started mongod.service - MongoDB Database Server.Sep 04 10:23:18 logging mongod[2338]: {"t":{"$date":"2024-09-04T10:23:18.570Z"},"s":"I",  "c":"CONTROL",  "id":7484500, "ctx":"main","msg":"Environment vari>
```

#### Opensearch

```jsx
sudo systemctl status opensearch
```

```jsx
● opensearch.service - OpenSearch
     Loaded: loaded (/lib/systemd/system/opensearch.service; enabled; preset: enabled)
     **Active: active (running) since Wed 2024-09-04 11:52:40 UTC; 20min ago**       Docs: <https://opensearch.org/>   Main PID: 3763 (java)
      Tasks: 88 (limit: 9358)
     Memory: 4.4G
        CPU: 52.860s
     CGroup: /system.slice/opensearch.service             └─3763 /usr/share/opensearch/jdk/bin/java -Xshare:auto -Dopensearch.networkaddress.cache.ttl=60 -Dopensearch.networkaddress.cache.negative.ttl=>Sep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager has been called by org.opensearch.bootstrap.OpenSearch (file:/usr/shar>Sep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: Please consider reporting this to the maintainers of org.opensearch.bootstrap.OpenSearchSep 04 11:52:30 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager will be removed in a future release
Sep 04 11:52:31 logging systemd-entrypoint[3763]: Sep 04, 2024 12:52:31 PM sun.util.locale.provider.LocaleProviderAdapter <clinit>Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: COMPAT locale provider will be removed in a future release
Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: A terminally deprecated method in java.lang.System has been called
Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager has been called by org.opensearch.bootstrap.Security (file:/usr/share/>Sep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: Please consider reporting this to the maintainers of org.opensearch.bootstrap.SecuritySep 04 11:52:31 logging systemd-entrypoint[3763]: WARNING: System::setSecurityManager will be removed in a future release
Sep 04 11:52:40 logging systemd[1]: Started opensearch.service - OpenSearch.
```

#### Graylog-server

```jsx
sudo systemctl status graylog-server
```

```jsx
● graylog-server.service - Graylog server
     Loaded: loaded (/lib/systemd/system/graylog-server.service; enabled; preset: enabled)
     **Active: active (running) since Wed 2024-09-04 12:09:22 UTC; 4min 56s ago**       Docs: <http://docs.graylog.org/>   Main PID: 5236 (graylog-server)
      Tasks: 131 (limit: 9358)
     Memory: 821.5M
        CPU: 45.499s
     CGroup: /system.slice/graylog-server.service             ├─5236 /bin/sh /usr/share/graylog-server/bin/graylog-server
             └─5237 /usr/share/graylog-server/jvm/bin/java -Xms1g -Xmx1g -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow -Djdk.tls.acknowledgeCloseNotif>Sep 04 12:09:22 logging systemd[1]: Started graylog-server.service - Graylog server.
```

#### Open Ports

If all the services are running, the following ports should be open

```jsx
sudo ss -plnt
```

```jsx
State       Recv-Q       Send-Q                           Local Address:Port              Peer Address:Port      Process
LISTEN      0            4096                                 127.0.0.1:27017                  0.0.0.0:*          users:(("mongod",pid=2338,fd=14))
LISTEN      0            4096                                         *:9200                         *:*          users:(("java",pid=3763,fd=636))
LISTEN      0            4096                                         *:9000                         *:*          users:(("java",pid=5237,fd=80))
LISTEN      0            4096                                         *:9300                         *:*          users:(("java",pid=3763,fd=634))
```

→ Check Login

Open up

```jsx
http://<server ip>:9000
```

Login with admin and the password you created earlier

(the one you entered at the password prompt not the sha string)

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/40235501-99bf-439a-9e07-e1d29132fcdd/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image.png)

image.png

Should login

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/f4e4352c-05a6-45cf-b603-a6acd7668511/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_1.png)

image.png

We will set up Graylog later

### → Graylogs log file

This caught me out because my mondod wasn’t running, and journalctl -u graylog-server didn’t give anything other than graylog had started but port 9000 wasn’t available.

The Log files for troubleshooting are

```jsx
sudo tail -f /var/log/graylog-server/server.log
```

Output

```jsx
2024-09-04T13:09:34.214+01:00 INFO  [MongoIndexSet] Index <gl-system-events_0> has been successfully allocated.2024-09-04T13:09:34.214+01:00 INFO  [MongoIndexSet] Pointing index alias <gl-system-events_deflector> to new index <gl-system-events_0>.2024-09-04T13:09:34.246+01:00 INFO  [MongoIndexSet] Successfully pointed index alias <gl-system-events_deflector> to index <gl-system-events_0>.2024-09-04T13:09:34.609+01:00 INFO  [NetworkListener] Started listener bound to [0.0.0.0:9000]
2024-09-04T13:09:34.610+01:00 INFO  [HttpServer] [HttpServer] Started.2024-09-04T13:09:34.611+01:00 INFO  [JerseyService] Started REST API at <0.0.0.0:9000>2024-09-04T13:09:34.611+01:00 INFO  [ServiceManagerListener] Services are healthy
2024-09-04T13:09:34.612+01:00 INFO  [ServerBootstrap] Services started, startup times in ms: {LocalKafkaMessageQueueReader [RUNNING]=0, UrlWhitelistService [RUNNING]=0, InputSetupService [RUNNING]=0, UserSessionTerminationService [RUNNING]=0, GracefulShutdownService [RUNNING]=0, LocalKafkaMessageQueueWriter [RUNNING]=1, FailureHandlingService [RUNNING]=1, PrometheusExporter [RUNNING]=1, OutputSetupService [RUNNING]=2, GeoIpDbFileChangeMonitorService [RUNNING]=2, BufferSynchronizerService [RUNNING]=4, EtagService [RUNNING]=5, StreamCacheService [RUNNING]=7, MongoDBProcessingStatusRecorderService [RUNNING]=12, LookupTableService [RUNNING]=12, NotificationSystemEventPublisher [RUNNING]=16, JobSchedulerService [RUNNING]=37, LocalKafkaJournal [RUNNING]=40, PeriodicalsService [RUNNING]=55, ConfigureCertRenewalJobOnStartupService [RUNNING]=79, JerseyService [RUNNING]=1351}
2024-09-04T13:09:34.612+01:00 INFO  [InputSetupService] Triggering launching persisted inputs, node transitioned from Uninitialized [LB:DEAD] to Running [LB:ALIVE]
2024-09-04T13:09:34.613+01:00 INFO  [ServerBootstrap] Graylog server up and running.
```

**Graylog** the service will start, however the application won’t until it sees **mongodb** and **opensearch** ports open

## Graylog configuration

We will use Filebeat to push the local logs from the “client” servers into the Graylog server, to do this, we need to create an input for Filebeat on Graylog.

### → Create Inputs

Within Graylog click on **System → Inputs**

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/104341ac-3ef1-4c4e-82c3-cdee1925a5a7/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_2.png)

image.png

Under **Select input** choose **Beats**

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/930819ec-e766-4256-86d3-12676a13e211/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_3.png)

image.png

Select **Launch New Input**

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/82d2e41d-6532-4d6f-9498-293a37b0db35/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_4.png)

image.png

In the Popup Give the Input a title

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/7b2c9d57-95ce-4771-863e-0db62355a636/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_5.png)

image.png

Scrill to the bottom and click on **Launch Input**

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/d0045606-3aa6-4c45-b441-6b67af6eddff/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_6.png)

image.png

This will start this input

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/845d8ee7-da79-455f-a70c-1ec246839db2/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_7.png)

image.png

If the command

```jsx
sudo s -plnt
```

Is run on the graylog server the port 5044 should be open

```jsx
LISTEN      0  4096    *:5044   *:*          users:(("java",pid=5237,fd=111))
```

Graylog is now installed and ready to receive data

## Client Installation - Filebeat

There are other options such as `systemd-journal-remote`/`systemd-journal-upload` I’ve used filebeat as I have used it on other servers in the past.

### → Install

Install required software

```jsx
sudo apt install sudo gnupg2 apt-transport-https curl -y
```

Import the key for elastic

```jsx
wget -qO - <https://artifacts.elastic.co/GPG-KEY-elasticsearch> | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/elastic.gpg
```

Add the elastic APT repo

```jsx
echo "deb <https://artifacts.elastic.co/packages/8.x/apt> stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.list
```

Update

```jsx
sudo apt update
```

Install Filebeat

```jsx
sudo apt install filebeat
```

Check version

```jsx
filebeat version
```

Returns something like:

```jsx
filebeat version 8.15.0 (amd64), libbeat 8.15.0 [76f45fe4dfkjdfjk36be495d2e1af08243 built 2024-08-02 17:15:22 +0000 UTC]
```

### → Configure

Edit the filebeat config

**Note: this is a YAML file so be aware of tabs and spaces**

```jsx
sudo nano /etc/filebeat/filebeat.yml
```

The following changes need to be made to this file

Filebeat Inputs

```jsx
# ============================== Filebeat inputs ===============================filebeat.inputs:# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.# Below are the input-specific configurations.# filestream is an input for collecting log messages from files.**- type: journald**  # Unique ID among all inputs, an ID is required.  **id: management-jorunald-input**  # Change to true to enable this input configuration.  **enabled: true**  # Paths that should be crawled and fetched. Glob based paths.  paths:    **- /var/log/journal**    #- c:\\programdata\\elasticsearch\\logs\\*
```

Comment out Elastic Search Output

```jsx
# ---------------------------- Elasticsearch Output ----------------------------#output.elasticsearch:  # Array of hosts to connect to. # hosts: [100.68.57.126:9200"]  # Performance preset - one of "balanced", "throughput", "scale",  # "latency", or "custom".  #preset: balanced
  # Protocol - either `http` (default) or `https`.  #protocol: "https"  # Authentication credentials - either API key or username/password.  #api_key: "id:api_key"  #username: "elastic"  #password: "changeme"
```

Edit Logstash Output

```jsx
# ------------------------------ Logstash Output -------------------------------**output.logstash:**  # The Logstash hosts
  **hosts: ["<graylog server>:5044"]**
```

Save and exit the file

### → Run

```jsx
sudo systemctl enable --now filebeat
```

Check its running

```jsx
sudo systemctl status filebeat
```

should return

```jsx
● filebeat.service - Filebeat sends log files to Logstash or directly to Elasticsearch.     Loaded: loaded (/lib/systemd/system/filebeat.service; disabled; preset: enabled)
     Active: active (running) since Wed 2024-09-04 12:40:23 UTC; 32s ago
       Docs: <https://www.elastic.co/beats/filebeat>   Main PID: 412066 (filebeat)
      Tasks: 8 (limit: 2234)
     Memory: 123.2M
        CPU: 9.211s
     CGroup: /system.slice/filebeat.service             └─412066 /usr/share/filebeat/bin/filebeat --environment systemd -c /etc/filebeat/filebeat.yml --path.home /usr/share/filebeat --path.config /et>
```

## Deploy Using Ansible

The following ansible play will deploy the filebeat to multple servers, keep both files in the same folder

### → Playbook - playbook-filebeat.yaml

```yaml
---- name: Install Filebeat on Debian 12  hosts: proxmox  become: yes  tasks:    - name: Ensure prerequisites are installed      apt:        name:          - gnupg2          - apt-transport-https          - curl        state: present    - name: Add Elastic GPG key      apt_key:        url: <https://artifacts.elastic.co/GPG-KEY-elasticsearch>        state: present    - name: Add Elastic APT repository      apt_repository:        repo: "deb <https://artifacts.elastic.co/packages/8.x/apt> stable main"        state: present        filename: 'elastic-8'    - name: Update APT package cache      apt:        update_cache: yes    - name: Install Filebeat      apt:        name: filebeat        state: present    - name: Copy new Filebeat configuration      copy:        src: filebeat.yml        dest: /etc/filebeat/filebeat.yml        owner: root        group: root        mode: '0644'    - name: Verify Filebeat configuration      command: filebeat test config      register: filebeat_config_test      changed_when: false    - name: Display Filebeat configuration test results      debug:        var: filebeat_config_test.stdout_lines    - name: Enable and start Filebeat service      systemd:        name: filebeat        enabled: yes        state: started    - name: Restart Filebeat service      systemd:        name: filebeat        state: restarted        enabled: yes
```

Change the line `hosts:` to relfect your inventory setup

### → filebeat.yml

```yaml
# ============================== Filebeat inputs ===============================filebeat.inputs:# Each - is an input. Most options can be set at the input level, so# you can use different inputs for various configurations.# Below are the input-specific configurations.# filestream is an input for collecting log messages from files.- type: journald  # Unique ID among all inputs, an ID is required.  id: journald-input  # Change to true to enable this input configuration.  enabled: true  # Paths that should be crawled and fetched. Glob based paths.  paths:    - /var/log/journal    #- c:\\programdata\\elasticsearch\\logs\\*# ============================== Filebeat modules ==============================filebeat.config.modules:  # Glob pattern for configuration loading  path: ${path.config}/modules.d/*.yml  # Set to true to enable config reloading  reload.enabled: false  # Period on which files under path should be checked for changes  #reload.period: 10s# ======================= Elasticsearch template setting =======================setup.template.settings:  index.number_of_shards: 1  #index.codec: best_compression  #_source.enabled: false# ------------------------------ Logstash Output -------------------------------output.logstash:  # The Logstash hosts  hosts: ["<server-ip>:5044"]  # Optional SSL. By default is off.  # List of root certificates for HTTPS server verifications  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]  # Certificate for SSL client authentication  #ssl.certificate: "/etc/pki/client/cert.pem"  # Client Certificate Key  #ssl.key: "/etc/pki/client/cert.key"# ================================= Processors =================================processors:  - add_host_metadata:      when.not.contains.tags: forwarded  - add_cloud_metadata: ~  - add_docker_metadata: ~  - add_kubernetes_metadata: ~
```

**Note:**

Remember to edit `hosts: ["<server-ip>:5044"]`

Put the IP of your graylog server

```yaml
ansible-playbook
```

### Graylog Server - Test Data

At this point, graylog is working and filebeat is setup, lets see what data we can view

### → View Data

Open the Graylog Web Portal

```jsx
http://<server ip>:9000/search
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/4143bf0d-0aab-4232-a6ad-f8a5121d96e5/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_8.png)

image.png

## Troubleshooting

### → Greylog server Java Ram

By default Graylog uses 1Gb or ram for the Java stack it runs on. to change this edit the file

```jsx
sudo nano /etc/default/graylog-server
```

Edit the line and change the `**Xms1g -Xmx1g**`

```jsx
GRAYLOG_SERVER_JAVA_OPTS="-**Xms4g -Xmx4g** -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow"
```

Restart the server

```jsx
sudo systemctl restart graylog-server
```

```jsx
http://<server ip>:9000/system/nodes
```

![](https://prod-files-secure.s3.us-west-2.amazonaws.com/d56e23bf-2ad0-456f-b113-a01e394b2b74/376d386c-26d9-421e-8290-388bcfffc8f6/Centralising_your_Journald_logs_using_Filebeat_and_ad47d520f62c4683b0beed413995bf33image_9.png)

image.png

### → Graylog is showing a system error about dropping elastic search messages.

When you fuirst start pointing all the nodes to graylog it gets a bit overwhelmed and starts alerting that its dropping messages before they are processed.

To fix this edit

```jsx
sudo nano /etc/graylog/server/server.conf
```

find

```jsx
#message_journal_max_age = 12h#message_journal_max_size = 5g
```

change them as apropriate

```jsx
message_journal_max_age = 6hmessage_journal_max_size = 15gb
```

Restart the server
