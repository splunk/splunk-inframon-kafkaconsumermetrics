# splunk-inframon-kafkaconsumermetrics
Docs to deploy the kafka consumer lag receiver for splunk inframon


## Purpose
This document is meant to walk you through the deployment of the kafkametrics receiver within a local branch of open-telemetry collector. The main benefit of this plugin is that it can read data from the topic ***__consumer_offsets*** to collect metrics related to the offset of all consumer groups connected to the broker. It is also able to fetch the latest offset of a topic partition that the broker is the leader for. Thus, its able to provide insight into consumer lag (or how far behind a consumer group is compared to the latest message on the topic partition)

You only need to run one kafkametrics receiver per cluster. Ideally, you add as many brokers as possible to the broker string in the configuration. The kafkametrics receiver will connect to the first available broker in the list, and then automatically discover the state of the cluster to fetch info from all the other connected brokers. The list of brokers is referenced to initiate the connection with an available broker on each collection cycle.

The minimum version of Apache Kafka required for scraping consumer lag is 0.10.0.0


We’ve merged the kafkametrics receiver in the otel-collector-contrib repo and the splunk-otel-collector repo [release](https://github.com/signalfx/splunk-otel-collector). 


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

You can grab the latest splunk-otel-collector [release](https://github.com/signalfx/splunk-otel-collector/releases) or run via one of the steps listed on the main [repo](https://github.com/signalfx/splunk-otel-collector) 


### Binary

```
./otelcol_linux_amd64 --config collector.yaml
```

### Docker (you need a local config file called collector.yaml)

```
docker run --rm -e SPLUNK_ACCESS_TOKEN=12345 -e SPLUNK_REALM=us0/us1/eu0 \
    -e SPLUNK_CONFIG=/etc/collector.yaml -v collector.yaml:/etc/collector.yaml:ro \
    --name otelcol quay.io/signalfx/splunk-otel-collector:0.26.0
```

### Configuration File
A reference yaml is provided here <Insert Link>

```
receivers:
  kafkametrics:
    protocol_version: 2.0.0
    brokers: 
      - "<broker-ip-1>:9092"
      - "<broker-ip-2>:9092"
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

In order to instantiate a dashboard, the simplest method would be to import [this](https://github.com/splunk/splunk-inframon-kafkaconsumermetrics/blob/main/SFx-Kafka-Lag-Dashboards.json) Dashboard Group into your org.

Alternatively, you can also leverage terraform. Once you’ve installed terraform these are the steps to follow:

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
