---
title: DC/OS
type: documentation
weight: 80
menu:
    main:
      parent: installation
---

# DC/OS

>**Note**: In this guide we will be using [DC/OS] (http://dcos.io) as our container-manager, but this guide will also work with Mesos and Marathon.

Mesos, Marathon and ZooKeeper are all installed by DC/OS, additionally Vamp requires Elasticsearch and Logstash for its metrics collection and aggregation.
One way to install Elasticsearch on DC/OS is by following the [Mesos Elasticsearch documentation](http://mesos-elasticsearch.readthedocs.org/en/latest/).
But we also need Logstash (which is not available as a package) and give it a [specific configuration] (https://github.com/magneticio/vamp-docker/blob/master/clique-base/logstash/logstash.conf), so we created a compatible Docker [images](https://hub.docker.com/r/magneticio/elastic/) that can be used with [How to install on Marathon](http://mesos-elasticsearch.readthedocs.org/en/latest/#how-to-install-on-marathon).
Our advice is to use our custom Elasticsearch+Logstash Docker image. Let’s go!

First, let's create `marathon.json` file with the following content:
{{% copyable %}}
```json
{
  "id": "elasticsearch",
  "container": {
    "docker": {
      "image": "mesos/elasticsearch-scheduler",
      "network": "HOST",
      "forcePullImage": true
    }
  },
  "args": [
    "--zookeeperMesosUrl", "zk://zk-1.zk:2181/mesos",
    "--elasticsearchDockerImage", "magneticio/elastic:2.2",
    "--elasticsearchRam", "1024",
    "--elasticsearchPorts", "9200,9300"
  ],
  "cpus": 0.2,
  "mem": 512.0,
  "env": {
    "JAVA_OPTS": "-Xms128m -Xmx256m"
  },
  "instances": 1
}
```
{{% /copyable %}}

This is quite similar to the normal Mesos Elasticsearch installation - notice that we use our custom Docker image `"--elasticsearchDockerImage", "magneticio/elastic:2.2"` and we also increased the amount of memory (by default it is 256MB). 
We also explicitly added the default 9200 and 9300 Elasticsearch ports by adding the additional argument: `"--elasticsearchPorts", "9200,9300"`. (There are many other different arguments which can be used as well if you need them.)

Following the Mesos Elasticsearch documentation now send `marathon.json` file to Marathon:

```
curl -k -XPOST -d @marathon.json -H "Content-Type: application/json" http://MARATHON_IP_ADDRESS:8080/v2/apps
```

Ok, now we are going to deploy Vamp.

>**Note**: Vamp is available as DC/OS Universe package and you can proceed with Vamp deployment using also DC/OS web UI.

Create `vamp.json` file with content:
{{% copyable %}}
```json
{
  "id": "vamp/vamp",
  "instances": 1,
  "cpus": 0.5,
  "mem": 1024,
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "magneticio/vamp:0.8.5",
      "network": "HOST",
      "forcePullImage": true
    }
  },
  "env": {
    "VAMP_REST_API_PORT": "9090",
    "VAMP_PERSISTENCE_KEY_VALUE_STORE_ZOOKEEPER_SERVERS": "zk-1.zk:2181",
    "VAMP_CONTAINER_DRIVER_MESOS_URL": "http://leader.mesos:5050",
    "VAMP_CONTAINER_DRIVER_MARATHON_URL": "http://marathon.mesos:8080",
    "VAMP_GATEWAY_DRIVER_LOGSTASH_HOST": "elasticsearch-executor.elasticsearch.mesos",
    "VAMP_PULSE_ELASTICSEARCH_URL": "http://elasticsearch-executor.elasticsearch.mesos:9200",
    "VAMP_LIFTER_VAMP_GATEWAY_AGENT_ENABLED": "true"
  }
}
```
{{% /copyable %}}

If you installed on different ports and/or hosts, make sure to update environment variables. For instance, they might look like this:

```json
"env": {
  "VAMP_REST_API_PORT": "9090",
  "VAMP_PERSISTENCE_KEY_VALUE_STORE_ZOOKEEPER_SERVERS": "10.240.0.3:2181",
  "VAMP_CONTAINER_DRIVER_MESOS_URL": "http://10.240.0.3:5050",
  "VAMP_CONTAINER_DRIVER_MARATHON_URL": "http://10.240.0.3:8080",
  "VAMP_GATEWAY_DRIVER_LOGSTASH_HOST": "10.240.0.4",
  "VAMP_PULSE_ELASTICSEARCH_URL": "http://10.240.0.4:9200",
  "VAMP_LIFTER_VAMP_GATEWAY_AGENT_ENABLED": "true"
}
```

Now send the request to Marathon:

```
curl -k -XPOST -d @vamp.json -H "Content-Type: application/json" http://MARATHON_IP_ADDRESS:8080/v2/apps
```

Marathon will now start deploying Vamp and if all arguments are set correctly you will notice that Vamp Gateway Agent will be also be deployed automatically.
What we did with the (`VAMP_LIFTER_VAMP_GATEWAY_AGENT_ENABLED`) setting is that we enabled Vamp to automatically deploy Vamp Gateway Agent’s on each Marathon agent(node). Handy!

Now we need to find out on what IP the public node is running, so you can access the VAMP dashboard from the outside. This IP is most often the same as where you find the Marathon dashboard. Vamp will expose its UI on port 9090 (`network: HOST`), you may set a different port (`VAMP_REST_API_PORT`) if you want.

Happy VAMP’ing! Make sure to run through our [QuickStart tutorials](http://vamp.io/documentation/guides/).
