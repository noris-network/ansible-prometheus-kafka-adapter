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
`prometheus_kafka_adapter_ssl_client_copy_files`|False|Set this to true in order to copy certifacte files over to the target.
`prometheus_kafka_adapter_ssl_client_cert_file`|""|Kafka client cert file that will be copied when `prometheus_kafka_adapter_ssl_client_copy_files` is `true`.
`prometheus_kafka_adapter_ssl_client_key_file`|""|Kafka client cert key file that will be copied when `prometheus_kafka_adapter_ssl_client_copy_files` is `true`.
`prometheus_kafka_adapter_ssl_ca_cert_file`|""|Kafka SSL Broker CA certificate file that will be copied when `prometheus_kafka_adapter_ssl_client_copy_files` is `true`.
`prometheus_kafka_adapter_ssl_client_key_pass`|""|Kafka client cert key password string.
`prometheus_kafka_adapter_ssl_client_cert`|"{{ prometheus_kafka_adapter_config_dir }}/client-cert.pem"|Location of the client certificate on the target. Set this to `""` if no client authentication is to be used.
`prometheus_kafka_adapter_ssl_client_key`|"{{ prometheus_kafka_adapter_config_dir }}/client-key.pem"|Location of the client certificate on the target. Set this to `""` if no client authentication is to be used.
`prometheus_kafka_adapter_ssl_ca_cert`|"{{ prometheus_kafka_adapter_config_dir }}/ca-cert.pem"|Location of the client certificate on the target. Set this to `""` if no client authentication is to be used.

## Dependencies

- [geerlingguy.docker](https://github.com/geerlingguy/ansible-role-docker)

## Example Playbooks

This playbook will use client certificate authentication and copy the necessary
certificates over.

```yaml
- hosts: servers
  become: true
  roles:
      - noris-network.ansible-prometheus-kafka-adapter
  vars:
    prometheus_kafka_adapter_kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:9093
    prometheus_kafka_adapter_kafka_topic: metrics.prometheus
    prometheus_kafka_adapter_ssl_client_copy_files: true
    prometheus_kafka_adapter_ssl_client_cert_file: files/certs/kafka/client-cert.pem
    prometheus_kafka_adapter_ssl_client_key_file: files/certs/kafka/client-key.pem
    prometheus_kafka_adapter_ssl_ca_cert_file: files/certs/kafka/ca-cert.pem
```

This playbook will use certificates inlined in a file encrypted by Ansible Vault but don't use the copy functionality provided by the role.

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
    prometheus_kafka_adapter_kafka_broker_list: kafka01:9093,kafka02:9093,kafka03:9093
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
    - noris-network.ansible-prometheus-kafka-adapter
```

[gin]: https://github.com/gin-gonic/gin
[pka]: https://github.com/Telefonica/prometheus-kafka-adapter