# Graylog Installation on Ubuntu 22.04
How to keep your infrastructure logs under control by setting up a Graylog service using a server with Nginx, Elasticsearch, and MongoDB

Step-by-step installation of the Graylog service on Ubuntu 22.04.

## Preparation

Update repositories:

```bash
  sudo apt update -y
  sudo apt upgrade
```
Install dependencies

```bash
  apt install apt-transport-https gnupg2 uuid-runtime pwgen curl dirmngr -y
```
## Instalar JDK Java

```bash
  apt install openjdk-11-jre-headless -y
  java -version
```
You should get this output: 

```bash
  openjdk version "11.0.24" 2024-07-16
  OpenJDK Runtime Environment (build 11.0.24+8-post-Ubuntu-1ubuntu322.04)
  OpenJDK 64-Bit Server VM (build 11.0.24+8-post-Ubuntu-1ubuntu322.04, mixed mode, sharing)

```
## Install and Configure Elasticsearch

Download and add the Elasticsearch GPG key with the following command:

```bash
  sudo su
  wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
```
Add the repository and install Elasticsearch:

```bash
  echo "deb https://artifacts.elastic.co/packages/oss-7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
```
Update the repository and install Elasticsearch:

```bash
  apt update -y
  apt install elasticsearch-oss -y
```
Now that Elasticsearch is installed, we need to edit the configuration file to define the cluster name:

```bash
  nano /etc/elasticsearch/elasticsearch.yml
```
Set the cluster name to Graylog and add another line as shown below:

```bash
  cluster.name: graylog
  action.auto_create_index: false
```

Save and close the file, then start the Elasticsearch service and enable it to start on boot with the following command:

```bash
  systemctl daemon-reload
  systemctl start elasticsearch
  systemctl enable elasticsearch
```

Check the service status with:

```bash
  systemctl status elasticsearch
```
You should see this output:

```bash
  elasticsearch.service - Elasticsearch
     Loaded: loaded (/usr/lib/systemd/system/elasticsearch.service; enabled; preset: enabled)
     Active: active (running) since Thu 2024-08-22 07:40:05 UTC; 19s ago
       Docs: https://www.elastic.co
   Main PID: 8481 (java)
      Tasks: 47 (limit: 2276)
     Memory: 1.2G (peak: 1.2G)
        CPU: 7.774s
     CGroup: /system.slice/elasticsearch.service
             └─8481 /usr/share/elasticsearch/jdk/bin/java -Xshare:auto -Des.networkaddress.cache.ttl=60 -Des.networkaddress.cache.negative.ttl=10 -XX:+AlwaysPreTouch -Xss1m -Djava.awt.headless=true -Dfile.>

ago 22 07:39:57 uri systemd[1]: Starting elasticsearch.service - Elasticsearch...
ago 22 07:40:05 uri systemd[1]: Started elasticsearch.service - Elasticsearch.
```
Now verify Elasticsearch's response with the following command:

```bash
  curl -X GET http://localhost:9200
```
You should get this output:

```bash
  {
  "name" : "uri",
  "cluster_name" : "graylog",
  "cluster_uuid" : "tWx823xKTMicIIGpLSveCw",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "oss",
    "build_type" : "deb",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

## Install MongoDB Server

Graylog uses MongoDB as its database, so we need to install MongoDB on the server. By default, the package is not included in Ubuntu's default repository, so we need to add it to the system:

You can add it with the following command:

```bash
  wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-4.4.list
```

Now that we have added it, update the repository cache and install Graylog:

```bash
  apt update -y
  apt install -y mongodb-org
```

In case of errors related to the libssl1.1 library during MongoDB installation, perform the following steps:

```bash
  echo "deb http://archive.ubuntu.com/ubuntu focal main" | sudo tee /etc/apt/sources.list.d/focal.list
  sudo apt update
  sudo apt install libssl1.1
  sudo rm /etc/apt/sources.list.d/focal.list
  sudo apt update
  sudo apt install -y mongodb-org
```

Start the service and check the status:

```bash
  systemctl enable --now mongod
  systemctl status mongod
```

The output should be as follows:

```bash
mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-08-22 08:33:24 UTC; 54s ago
       Docs: https://docs.mongodb.org/manual
   Main PID: 8414 (mongod)
     Memory: 60.7M
        CPU: 867ms
     CGroup: /system.slice/mongod.service
             └─8414 /usr/bin/mongod --config /etc/mongod.conf

ago 22 08:33:24 uri systemd[1]: Started MongoDB Database Server.
ago 22 08:33:24 uri mongod[8414]: {"t":{"$date":"2024-08-22T08:33:24.363Z"},"s":"I",  "c":"CONTROL",  "id":7484500, "ctx":"main","msg":"Environment variable MONGODB_CONFIG_OVERRIDE_NOFORK == 1, overriding >
```

## Install and Configure Graylog

We need to add the Graylog repository to the server and install it.

```bash
  wget https://packages.graylog2.org/repo/packages/graylog-4.3-repository_latest.deb
```
Once the package is downloaded, install it:

```bash
  dpkg -i graylog-4.3-repository_latest.deb
  apt update -y
  apt install graylog-server -y
```
Now we need to create a secret to secure user passwords:

```bash
  pwgen -N 1 -s 96
```
Next, we need to generate another password for the Graylog admin user, which we will need to access the Graylog web interface.

```bash
  echo -n "Enter Password: " && head -1
```

Now edit the Graylog config file and define both passwords:

```bash
  nano /etc/graylog/server/server.conf
```

Edit the file as follows:

```bash
  password_secret = Wv4VQWCAA9sRbL7pxPeY7tb9lSo50esEWgNXxXHypx0Og3CezMmQLdF2QzQdRSIXmNXKINjRvZpPTrvZv4k4NlJrFYTfOc3c
  root_password_sha2 = e472e1436cbe87774c1bc75d0a646d67e506bea1dff8701fd41f34bca33e1419   
```

Set your IP address in the Graylog configuration file:

```bash
  http_bind_address = 10.0.26.71:9000
```

Save the file and start the Graylog service:

```bash
  systemctl daemon-reload
  systemctl start graylog-server
  systemctl enable graylog-server
  systemctl status graylog-server
```

When checking the status, you should see the following output:

```bash
graylog-server.service - Graylog server
     Loaded: loaded (/lib/systemd/system/graylog-server.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2024-08-22 10:23:21 UTC; 1min 4s ago
       Docs: http://docs.graylog.org/
   Main PID: 9344 (graylog-server)
      Tasks: 116 (limit: 5967)
     Memory: 798.1M
        CPU: 25.748s
     CGroup: /system.slice/graylog-server.service
             ├─9344 /bin/sh /usr/share/graylog-server/bin/graylog-server
             └─9381 /usr/bin/java -Xms1g -Xmx1g -XX:NewRatio=1 -server -XX:+ResizeTLAB -XX:-OmitStackTraceInFastThrow -Djdk.tls.acknowledgeCloseNotify=true -Dlog4j2.formatMsgNoLookups=true -XX:+UseConcMark>

ago 22 10:23:21 uri systemd[1]: Started Graylog server.
```

Now the service should be running and listening on port 9000. Let's check it:

```bash
  ss -antpl | grep 9000
```

Return the following:

```bash
  LISTEN 0      4096   [::ffff:127.0.0.1]:9000             *:*    users:(("java",pid=9381,fd=79)) 
```

## Configure Nginx as a reverse proxy for Graylog

We install the Nginx service:

```bash
  apt install nginx -y
```

We create a virtual host configuration file for Nginx with the following command:

```bash
  nano /etc/nginx/sites-available/graylog.conf
```

And we add the following lines:

```bash
  server {
    listen 80;
    server_name graylog.example.org;

  location /
    {
      proxy_set_header Host $http_host;
      proxy_set_header X-Forwarded-Host $host;
      proxy_set_header X-Forwarded-Server $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Graylog-Server-URL http://$server_name/;
      proxy_pass       http://208.117.84.72:9000;
    }

  }
```

Save the file and run the following command to ensure that the syntax of the file is correct:

```bash
  nginx -t
```

We enable the host configuration file with the following command:

```bash
  ln -s /etc/nginx/sites-available/graylog.conf /etc/nginx/sites-enabled/
```

We remove the default Nginx virtual host file:

```bash
  rm -rf /etc/nginx/sites-enabled/default
```

Then restart the Nginx service:

```bash
  systemctl restart nginx
  systemctl status nginx
```

## Access the Graylog environment from the web interface

In our browser, enter http://ip_address:9000

It will prompt us to log in. The default username is admin, and the password is the one we set earlier in SHA-256.
