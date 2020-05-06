# prometheus-kafka-adapter

This role downloads and launches the [prometheus-kafka-adapter][pka]
as a Docker container. The container will be managed by systemd on the target
host.

## Requirements

- Ansible >= 2.7 (It might work on previous versions, but we cannot guarantee it)

## Role Variables

All variables which can be overridden are stored in
[defaults/main.yml](defaults/main.yml) file as well as in table below. For more information on the configuration parameters please
refer to the official [Github repo][pka].

Name|Default Value|Description
---|---|---
`prometheus_kafka_adapter_config_dir`|/etc/prometheus-kafka-adapter|The config dir on the target host.
`prometheus_kafka_adapter_docker_image`|telefonica/prometheus-kafka-adapter:1.6.0|The Docker image to use for the adapter.
`prometheus_kafka_adapter_container_name`|prometheus-kafka-adapter|The name of the container to be run on the target host.
`prometheus_kafka_adapter_kafka_broker_list`|kafka1:9092,kafka2:9092|A list of Kafka brokers to send the data to.
`prometheus_kafka_adapter_kafka_topic`|metrics|The Kafka topic to send the data to.
`prometheus_kafka_adapter_kafka_compression`|none|The compression type to be used.
`prometheus_kafka_adapter_kafka_batch_num_messages`|10000|The number of batches to write.
`prometheus_kafka_adapter_kafka_serialization_format`|json|Defines the serialization format (`json` or `avro-json`)
`prometheus_kafka_adapter_listen_port`|8080|The HTTP port to listen on.
`prometheus_kafka_adapter_log_level`|info|The log level of prometheus-kafka-adapter.
`prometheus_kafka_adapter_gin_mode`|release|The [gin][gin] log level.
`prometheus_kafka_adapter_ssl_client_cert_file`|""|Kafka client cert file.
`prometheus_kafka_adapter_ssl_client_key_file`|""|Kafka client cert key file.
`prometheus_kafka_adapter_ssl_client_key_pass`|""|Kafka client cert key password string.
`prometheus_kafka_adapter_ssl_ca_cert_file`|""|Kafka SSL Broker CA certificate file.

## Dependencies

- [geerlingguy.docker](https://github.com/geerlingguy/ansible-role-docker)

## Example Playbook

```yaml
- hosts: servers
  become: true
  roles:
      - noris-network.ansible-prometheus-kafka-adapter
  vars:
    prometheus_kafka_adapter_kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:9093
    prometheus_kafka_adapter_kafka_topic: metrics.prometheus
    prometheus_kafka_adapter_ssl_client_cert_file: files/certs/kafka/client-cert.pem
    prometheus_kafka_adapter_ssl_client_key_file: files/certs/kafka/client-key.pem
    prometheus_kafka_adapter_ssl_ca_cert_file: files/certs/kafka/ca-cert.pem
```

[gin]: https://github.com/gin-gonic/gin
[pka]: https://github.com/Telefonica/prometheus-kafka-adapter