# Central Logging
Setup with Graylog, OpenSearch, and Filebeat all running in Docker containers.

The webinterfaces of Graylog and OpenSearch-Dashboard will be available through the reverse proxy Traefik.

## Goal
The goal is to gather logs from various servers on a central Logging server.
In particular, logs from docker containers and pods in a Kubernetes cluster should be available in a web dashboard.

## Components

![Components](diagrams/central-logging-components.jpg)


## Central Logging Server

In this section we set up the central logging server. 
It serves following services:
- Graylog
- OpenSearch
- OpenSearch-Dashboard
- Mongo
- cert-extract - make acme certificates from Traefik available in Graylog
- Traefik - reverse proxy

The setup is tested using Ubuntu 22.04 - but it should work on other Linux as well.

### Prerequisites
- Subdomain for Graylog
- Subdomain for OpenSearch-Dashboard
- Server

### Setup

Open ports mentioned in `docker-compose.host.yml` and `docker-compose.traefik.yml` in your firewall. For more details, have a look at the section [VM Security Groups](#vm-security-groups). 
If you are using a simple software firewall like Firewalld or UFW, keep in mind that [docker bypasses all firewall rules](https://dev.to/soerenmetje/how-to-secure-a-docker-host-using-firewalld-2joo) in default configuration.

Create DNS Entries for subdomains for `graylog` and `opensearch-dashboard` service.

Clone this repository on your node and `cd` into the directory.

Start reverse proxy Traefik if it is not already up and running on the node:

```shell
docker compose -f docker-compose.traefik.yml up -d
```

Replace placeholder domain for `graylog` service with your subdomain in all files:
```shell
find . -type f -exec sed -i 's/logs.placeholderdomain.com/logs.mydomain.com/g' {} +
```

> The domain for graylog is used in multiple files such as `filebeat.yml` `docker-compose.host.yml` and `kubernetes-deploy/filebeat-daemonset.yaml`

Replace placeholder domain for `opensearch-dashboard` service with your subdomain in all files:
```shell
find . -type f -exec sed -i 's/os.placeholderdomain.com/os.mydomain.com/g' {} +
```

Copy `.env.graylog.template` and rename to `.env.graylog`:

```shell
cp .env.graylog.template .env.graylog
```

Set variables `GRAYLOG_ROOT_PASSWORD_SHA2` and `GRAYLOG_PASSWORD_SECRET` in `.env.graylog`.

Set path to `acme.json` (`letsencrypt` dir in traefik) in `cert-extract` service in `docker-compose.host.yml`. To do so you may search for `CHANGE PATH TO acme.json dir`

Start services:
```shell
docker compose -f docker-compose.host.yml up -d
```

The Graylog and OpenSearch-Dashboard webinterface should be available 
within 2 minutes.

Set auth in OpenSearch-Dashboard webinterface.
The default username and password is `admin`.

### VM Security Groups
In order to restrict access to the VM security groups can be used. 
To allow web browsers and log agents like filebeat to access Graylog, 
some security groups have to be added to host VM:

- 80 (http)
- 443 (https)
- 5044
- 1514
- 12201

> You also may want to restrict access for certain subnet or hosts

### Certificates
To enable TLS for `Inputs`, certificates are needed. 
In this setup, these can be extracted from Traefik. 
For that following image is used in `docker-compose.host.yml`: https://hub.docker.com/r/soerenmetje/acme-certs-extract 
The volume `./certs` is used for sharing certificates.
In Graylog, go to Input configuration and set *TLS cert file* to `/usr/share/graylog/certs/flat/logs.placeholderdomain.com_full.crt`
and *TLS private key file* to `/usr/share/graylog/certs/flat/logs.placeholderdomain.com.key`.

### Plugins
Graylog plugins can be used too. 
Put the plugin `.jar` files in the `./plugins` directory.
The directory `./plugins` is mapped by a volume into the Graylog container.

#### Nice Plugins
- Graylog plugin for Telegram alert notifications: [TelegramAlert](https://github.com/irgendwr/TelegramAlert)

## Agents
- System Logs: Syslog
- Docker Logs: FileBeat

### Syslog
Syslog can be used to gather System logs.

When starting with plain Graylog, create a `Input` for Syslog first. This is an input type not a single machine.

Configure the machine to push the logs to the configured Graylog input



Create file `/etc/rsyslog.d/graylog.conf` and add following content:
```
*.* @@logs.placeholderdomain.com:1514;RSYSLOG_SyslogProtocol23Format
# @@ = TCP 
# currently unsecure
```

Reload config:
```shell
sudo systemctl restart rsyslog.service 
sudo systemctl status rsyslog.service 
```

### FileBeat
FileBeat is a log-shipper and can be used to gathering logs from f.e. Docker.

When starting with plain Graylog, create a `Input` for FileBeat first. This is a general input for multiple FileBeat services.

#### FileBeat to Ship Container Logs in Docker
Change hosts in `filebeat.yml` to your graylog subdomain, if not already done.

Start FileBeat:
```shell
docker compose -f docker-compose.filebeat.yml up -d
```

#### FileBeat to Ship Container Logs in Kubernetes
Change hosts in `kubernetes-deploy/filebeat-daemonset.yaml` to your graylog subdomain, if not already done.

Start FileBeat:
```shell
kubectl apply -f ./kubernetes-deploy
```

> Namespace is set to `kube-system` in yaml-files

## Sources
- Graylog Docs Docker https://go2docs.graylog.org/5-0/downloading_and_installing_graylog/docker_installation.htm
- Graylog & co @ Docker https://docs.graylog.org/docs/docker
- Filebeat @ Docker gathering Docker logs: https://www.sarulabs.com/post/5/2019-08-12/sending-docker-logs-to-elasticsearch-and-kibana-with-filebeat.html
- Filebeat @ Docker gathering system logs: https://www.elastic.co/guide/en/beats/filebeat/7.17/running-on-docker.html
- ELK @ Docker: https://medium.com/geekculture/shoving-your-docker-container-logs-to-elk-made-simple-882bffdbcad6
- Secure communication with Logstash: https://www.elastic.co/guide/en/beats/filebeat/current/configuring-ssl-logstash.html
- Graylog OpenSearch: https://go2docs.graylog.org/5-0/planning_your_deployment/upgrading_to_opensearch_-_installation.htm
