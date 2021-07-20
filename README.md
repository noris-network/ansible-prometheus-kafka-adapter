# prometheus-kafka-adapter

This role downloads and launches multiple [Telefonica
prometheus-kafka-adapter](https://github.com/Telefonica/prometheus-kafka-adapter)
(PKA) as a docker container. The container will be managed by systemd on the
target host.

The role is tested with vagrant:

- Centos 7
- Centos 8
- Ubuntu 18.04
- Ubuntu 20.04

Currently not working on vagrant for RHEL 7+8 and Debian 9+10

## Requirements

Ansible >= 2.7 (It might work on previous versions, but we cannot guarantee
  it)

## Role Variables

All variables which can be overridden are stored in
[defaults/main.yml](defaults/main.yml) file as well as in table below. For more
information on the configuration parameters please refer to the official [Github
repo](https://github.com/Telefonica/prometheus-kafka-adapter).

Name|Default Value|Description
---|---|---
`prometheus_kafka_adapter_config_dir`|/etc/prometheus-kafka-adapter|The config dir on the target host.
`prometheus_kafka_adapter_docker_registry`|none|custom docker registry, no http/https or tailing slash, defaults to `registry.hub.docker.com`
`prometheus_kafka_adapter_docker_username`|none|registry login username
`prometheus_kafka_adapter_docker_password`|none|registry user password
`prometheus_kafka_adapter_docker_image`|telefonica/prometheus-kafka-adapter:1.6.0|The Docker image to use for the adapter.
`prometheus_kafka_adapter_container_name`|prometheus-kafka-adapter|The name of the container to be run on the target host.
`prometheus_kafka_adapter_config_list`|[]|A list of prometheus-kafka-instances, for example one PKA per kafka topic.
`prometheus_kafka_adapter_kafka_compression`|none|The compression type to be used.
`prometheus_kafka_adapter_kafka_batch_num_messages`|10000|The number of batches to write.
`prometheus_kafka_adapter_kafka_serialization_format`|json|Defines the serialization format (`json` or `avro-json`)
`prometheus_kafka_adapter_listen_port`|8080|The HTTP port to listen on.
`prometheus_kafka_adapter_log_level`|info|The log level of prometheus-kafka-adapter.
`prometheus_kafka_adapter_gin_mode`|release|The [gin](https://github.com/gin-gonic/gin) log level.

> TODO: decide if we copy files or take content from vault for SSL certs

## Configure kafka adapter instances

To configure multiple kafka adapters on the same host, you have to define a list
of dictionaries that contains the configuration fields for your kafka instances.
Note that all kafka adpaters will share the same SSL certificates.

The following variable defines two adapters:

```yaml
prometheus_kafka_adapter_config_list:
  - name: metrics
    listen_port: 8080
    log_level: info
    gin_mode: release
    kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:9093
    kafka_topic: prometheus.metrics
  - name: othermetrics
    listen_port: 8081
    log_level: debug
    gin_mode: debug
    kafka_broker_list: kafka04:9093,kafka05:9093,kafka06:9093
    kafka_topic: prometheus.othermetrics
```

In case you don't specify these fields, the default values from `defaults/main.yml`
will be applied.

Name|Default Value|Description
---|---|---
`name`|"""|The name of PKA instance, appended to `prometheus_kafka_adapter_container_name`
`listen_port`|8080|The HTTP port to listen on.
`log_level`|info|The log level of prometheus-kafka-adapter.
`gin_mode`|release|The [gin](https://github.com/gin-gonic/gin) log level.
`kafka_broker_list`|kafka1:9092,kafka2:9092|A list of Kafka brokers to send the data to.
`kafka_topic`|metrics|The Kafka topic to send the data to.

## Remote write

When setting an url for `prometheus_remote_write` you can access an adapter's
listen port through its position in `prometheus_kafka_adapter_config_list`.

```yaml
prometheus_remote_write:
  - url: "http://{{ ansible_fqdn }}:{{ prometheus_kafka_adapter_config_list.0.listen_port }}/receive"
    write_relabel_configs:
      - source_labels: [__name__]
        regex: (?i)(.*some:metric:.+)
        action: keep
  - url: "http://{{ ansible_fqdn }}:{{ prometheus_kafka_adapter_config_list.1.listen_port }}/receive"
    write_relabel_configs:
      - source_labels: [__name__]
        regex: (?i)(.*some:metric:.+)
        action: keep
```

## Dependencies

- [geerlingguy.docker](https://github.com/geerlingguy/ansible-role-docker)

## Example Playbook

This playbook will use client certificate authentication and copy the necessary
certificates over.

```yaml
- name: Enable client auth
  hosts: servers
  become: true
  vars:
    prometheus_kafka_adapter_config_dir: "/etc/prometheus-kafka-adapter"
    prometheus_kafka_adapter_ssl_client_copy_files: false
    prometheus_kafka_adapter_ssl_client_cert: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.cert.name }}"
    prometheus_kafka_adapter_ssl_client_key: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.key.name }}"
    prometheus_kafka_adapter_ssl_ca_cert: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.ca.name }}"
    prometheus_kafka_adapter_kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:909
    prometheus_kafka_adapter_kafka_topic: metrics.prometheus
  tasks:
  - name: Ensure directory exists
    file:
      path: "{{ prometheus_kafka_adapter_config_dir }}"
      state: directory
      owner: root
      group: root
      mode: 0644
  - name: Copy client cert
    copy:
      content: "{{ vault.certificates.cert.content }}"
      dest: "{{ prometheus_kafka_adapter_ssl_client_cert }}"
      owner: root
      group: root
      mode: '0600'
  - name: Copy client cert key
    copy:
      content: "{{ vault.certificates.content }}"
      dest: "{{ prometheus_kafka_adapter_ssl_client_key }}"
      owner: root
      group: root
      mode: '0600'
  - name: Copy ca
    copy:
      content: "{{ vault.certificates.content }}"
      dest: "{{ prometheus_kafka_adapter_ssl_ca_cert }}"
      owner: root
      group: root
      mode: '0600'
```

This playbook will use certificates inlined in a file encrypted by Ansible Vault
but don't use the copy functionality provided by the role.

```yaml
- name: Install prometheus-kafka-adapter
  become: true
  hosts: servers
  vars:
    prometheus_kafka_adapter_config_dir: "/etc/prometheus-kafka-adapter"
    prometheus_kafka_adapter_ssl_client_cert: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.cert.name }}"
    prometheus_kafka_adapter_ssl_client_key: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.key.name }}"
    prometheus_kafka_adapter_ssl_ca_cert: "{{ prometheus_kafka_adapter_config_dir }}/{{ vault.certificates.ca.name }}"
    prometheus_kafka_adapter_kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:9093
    prometheus_kafka_adapter_kafka_topic: metrics.prometheus
  roles:
    - noris-network.prometheus-kafka-adapter
```

## License

This project is licensed under Apache 2.0 License. See [LICENSE](LICENSE) for more details.

## Authors

- [Alexander Knipping](https://github.com/obitech)
- [Daniel Paufler](https://github.com/egmont1227)
