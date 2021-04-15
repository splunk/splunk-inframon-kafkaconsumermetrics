# splunk-inframon-kafkaconsumermetrics
Docs to deploy the kafka consumer lag receiver for splunk inframon


## Purpose
This document is meant to walk you through the deployment of the kafka metrics receiver within a local branch of open-telemetry collector. The main benefit of this plugin is that it can read data from the topic ***__consumer_offsets*** to collect metrics related to the offset of all consumer groups connected to the broker. It is also able to fetch the latest offset of a topic partition that the broker is the leader for. Thus, its able to provide insight into consumer lag (or how far behind a consumer group is compared to the latest message on the topic partition)


The main branch of the open-telemetry collector usually takes a much longer time to accept PRs, and we’ve merged the receiver in the otel-collector-contrib repo. Splunk is also in the process of migrating all signalfx smart-agent monitors into the splunk otel collector agent. You can choose to wait until this receiver gets merged with the splunk otel collector agent to replace your smart-agent OR you can leverage a binary/docker container to just collect kafka consumer lag related metrics.

## Metrics Collection - Description + Details

### Topic related metrics collected by the go kafka client (Sarama by shopify)
* *kafka.topic.partitions* [# of partitions in topic, dims: topic] 
* *kafka.partition.current_offset* [Current offset of partition, dims: topic,partition] 
* *kafka.partition.oldest_offset* [Oldest offset of partition, dims: topic,partition] 
* *kafka.partition.replicas* [# of replicas for partition, dims: topic, partition] 
* *kafka.partition.replicas_in_sync* [# of synced replicas of partition, dims: topic,partition] 

### Consumer group related metrics collected by reading from the __consumer_offsets topic
* *kafka.consumer_group.members* [Count of members in consumer group, dims: group] 
* *kafka.consumer_group.offset* [Current offset of consumer group at partition, dims: group, topic, * *partition] 
* *kafka.consumer_group.lag* [Current approx lag of consumer group at partition, dims: group, * topic, *partition] 
* *kafka.consumer_group.lag_sum* [Current approx sum of consumer group lag across all partitions of * *topic, dims: group, topic] 
* *kafka.consumer_group.offset_sum* [Sum of consumer group offset across partitions, dims: group, topic] 

### Broker metrics - each metric timeseries identifies a broker
* *kafka.brokers* [# of brokers in the cluster] 


## Setup

You can grab the latest otel-collector-contrib [release](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases) or run as a dockerized container 


### Binary

```
./otelcol_linux_amd64 --config collector.yaml
```

### Docker (Assuming this is merged into splunk-otel-collector)

```
docker run --rm -e SPLUNK_ACCESS_TOKEN=12345 -e SPLUNK_REALM=us0/us1/eu0 \
    -e SPLUNK_CONFIG=/etc/collector.yaml -v collector.yaml:/etc/collector.yaml:ro \
    --name otelcol quay.io/signalfx/splunk-otel-collector:latest
```

### Configuration File
A reference yaml is provided here <Insert Link>

```
receivers:
  kafkametrics:
    protocol_version: 2.0.0
    brokers: <broker-ip>:9092
    collection_interval: 10s
    scrapers:
      - brokers
      - topics
      - consumers

exporters:
  signalfx:
    access_token: <insert-token>
    realm: us0/us1/eu0


service:
  pipelines:
    metrics:
      receivers: [kafkametrics]
      exporters: [signalfx]
```

For more configuration options, please visit [here](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/kafkametricsreceiver) 

## Dashboard

In order to instantiate a dashboard, the simplest method would be to leverage terraform. Once you’ve installed terraform these are the steps to follow:

```
#Clone our jumpstart repo 
git clone https://github.com/splunk/splunk-inframon-jumpstart.git

#Initialize Terraform
terraform init --upgrade

#Navigate to the cloned directory and review the execution plan
terraform plan -var="access_token=abc123" -var="realm=us0/us1/eu0" -target=module.kafka

#Apply the changes
terraform apply -var="access_token=abc123" -var="realm=us0/us1/eu0" -target=module.kafka

#[Optional] Destroy the changes if you do not need a reference dashboard/detectors
terraform destroy -var="access_token=abc123" -var="realm=us0/us1/eu0" -target=module.kafka
```

Alternatively, you can import [this](https://github.com/splunk/splunk-inframon-kafkaconsumermetrics/blob/main/SFx-Kafka-Lag-Dashboards.json) Dashboard Group into your org.
