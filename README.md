# Monitoring Stack
Full stack of tools for monitoring.

This stack is composed by:

- [Netdata](https://github.com/firehol/netdata):
`Real Time Monitoring` - Visualise real time system status
- [Prometheus](https://prometheus.io/)
`Data Base` - Store collected metrics
- [Docker](https://www.docker.com/)
`Container` - Base container solution
- [cAdvisor](https://github.com/google/cadvisor)
`Container metrics exporter` - Expose metrics of your running containers
- [Grafana](https://grafana.com/)
`Analytics plataform` - Allow you query and understand collected metrics
- [Node_Exporter](https://github.com/prometheus/node_exporter)
`OS & Hardware metrics exporter` - Expose OS & Hardware metrics
- [AlertManager](https://github.com/prometheus/alertmanager)
`Alerting` - Handle alerts sent by Prometheus
- [Slack](https://slack.com/)
`Communications` - Easy integration between your teammates

# Requirements

To execute the steps bellow the following are necessary: 

 - Docker in [Swarm](https://docs.docker.com/engine/swarm/) mode

You can simply run the followin to start your standalone [swarm cluster](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/):

```
$ docker swarm init --advertise-addr 192.168.99.100
Swarm initialized: current node (dxn1zf6l61qsb1josjja83ngz) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c \
    192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

# Setup

## Cloning this repo
 
```
cd
# git clone git@github.com:jeskz0rd/monitoring.git
cd monitoring
```

##  Netdata Installation steps

[Install Netdata](https://github.com/firehol/netdata/#installation):
```
# bash <(curl -Ss https://my-netdata.io/kickstart.sh) all
```

### Setup memory de-duplication

[Memory de-duplication](https://github.com/firehol/netdata/wiki/Memory-Deduplication---Kernel-Same-Page-Merging---KSM)

```
# echo 1 >/sys/kernel/mm/ksm/run
# echo 1000 >/sys/kernel/mm/ksm/sleep_millisecs
```

### Setup Netdata Scraper in Prometheus:

```
# vim /conf/prometheus/prometheus.yml
...
- job_name: 'netdata'
    metrics_path: '/api/v1/allmetrics'
    params:
      format: [prometheus]
    honor_labels: true
    scrape_interval: 20s
    static_configs:
         - targets: ['YOUR_NETDATA_IP:19999']
```

### Setup Node Labels

Add the "prom" label to the Prometheus Swarm node.

```
docker node update YOUR_PROMETHEUS_SWARM_NODE --label-add "prom=true"
```

### Setup Grafana Permissions

In Grafana 5.1> the default user id is 472 and as described in [Grafana Documentation](http://docs.grafana.org/installation/docker/) the steps bellow must be done to run it properly. 


```
# docker container run --rm --user root --name grafana_temp -it -v ~/monitoring/volumes/grafana/data:/var/lib/grafana --entrypoint bash grafana/grafana:5.1.3
```

yet in the container you just started run the following:

```
$ chown -R root:root /etc/grafana && \
chmod -R a+r /etc/grafana && \
chown -R grafana:grafana /var/lib/grafana && \
chown -R grafana:grafana /usr/share/grafana && \
exit
``` 

it takes a while changing the permissions...

### Deploy the stack

```
# docker stack deploy -c docker-compose.yml monitoring
```

Check deployed services:
```
# docker service ls

ID                  NAME                     MODE                REPLICAS            IMAGE                                   PORTS
ypjvzrdzs760        monitoring_alertmanager    replicated          1/1                 jesk/alertmanager_alpine:1.0    *:9093->9093/tcp
nnpqv7k297g4        monitoring_cadvisor        global              1/1                 google/cadvisor:v0.30.0         *:8080->8080/tcp
gpn2qklfmra6        monitoring_grafana         replicated          1/1                 grafana/grafana:5.1.3           *:3000->3000/tcp
31q4t856ciua        monitoring_node-exporter   global              1/1                 jesk/node-exporter_alpine:1.0   *:9100->9100/tcp
z2jd4eprumd8        monitoring_prometheus      replicated          1/1                 jesk/prometheus_alpine:1.0      *:9090->9090/tcp
```

Accessing Prometheus interface on browser:
```
http://YOUR_HOST_IP:9090
```

Accessing AlertManager interface on browser:
```
http://YOUR_HOST_IP:9093
```

Accessing Grafana interface on browser:
```
http://YOUR_HOST_IP:3000
user: admin
passwd: admin
```

Accessing Netdata interface on browser:
```
http://YOUR_HOST_IP:19999
```

Accessing Node_exporter metrics on browser:
```
http://YOUR_HOST_IP:9100/metrics
```

Create a channel and add the [API](https://get.slack.help/hc/en-us/articles/215770388-Create-and-regenerate-API-tokens) information about your [Slack account](https://api.slack.com/tokens)

```
# vim /conf/alertmanager/config.yml

route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            username: 'YOUR USERNAME'
            channel: '#YOURCHANNEL'
            api_url: 'INCOMING WEBHOOK'
``` 

# Changelog
All notable changes to this project will be documented in this [file](./CHANGELOG.md).