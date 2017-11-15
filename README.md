# Role Name

An [Ansible] role to install/configure [fluentd]

## Requirements

None

## Role Variables

```yaml
---
# defaults file for ansible-fluentd
fluentd_debian_repo_info:
  repo: 'deb http://packages.treasuredata.com/2/{{ ansible_distribution|lower }}/{{ ansible_distribution_release|lower }}/ {{ ansible_distribution_release|lower }} contrib'
  repo_key: 'http://packages.treasuredata.com/GPG-KEY-td-agent'

fluentd_inputs: []
  # - type: 'tail'
  #   format: 'apache2'
  #   tag: 'apache.access'
  #   path: '/var/log/apache2/access.log'
  # - type: 'tail'
  #   format: '/^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\] \[pid (?<pid>[^\]]*)\] \[client (?<client>[^\]]*)\] (?<message>.*)$/'
  #   tag: 'apache.error'
  #   path: '/var/log/apache2/error.log'
  # - type: 'syslog'
  #   port: 5140
  #   bind: 0.0.0.0
  #   tag: 'syslog'
  #   protocol_type: 'udp'
  # - type: 'tail'
  #   path: '/var/log/syslog'
  #   pos_file: '/var/log/syslog.pos'
  #   tag: 'syslog'
  #   format: 'syslog'

fluentd_network_optimizations:
  - name: 'net.ipv4.tcp_tw_recycle'
    state: 'present'
    value: 1
  - name: 'net.ipv4.tcp_tw_reuse'
    state: 'present'
    value: 1
  - name: 'net.ipv4.ip_local_port_range'
    state: 'present'
    value: '10240 65535'

fluentd_ouputs: []
  # - type: 'elasticsearch'
  #   logstash_format: true
  #   match: '**'
  #   host: '192.168.250.10'
  #   port: 9200
  #   index_name: 'fluentd'
  #   type_name: 'fluentd'
  # - type: 'rabbitmq'
  #   match: '**'
  #   host: 192.168.250.100
  #   user: 'graylog'
  #   pass: 'graylog'
  #   vhost: '/'
  #   format: 'json'
  #   exchange: 'graylog'
  #   exchange_type: 'direct'
  #   exchange_durable: true
  #   routing_key: 'graylog'
  #   heartbeat: 10

fluentd_plugins: []
  # - 'fluent-plugin-elasticsearch'
  # - 'fluent-plugin-rabbitmq'

fluentd_ulimits:
  - domain: 'root'
    limit_type: 'soft'
    limit_item: 'nofile'
    value: 65536
  - domain: 'root'
    limit_type: 'hard'
    limit_item: 'nofile'
    value: 65536
  - domain: '*'
    limit_type: 'soft'
    limit_item: 'nofile'
    value: 65536
  - domain: '*'
    limit_type: 'hard'
    limit_item: 'nofile'
    value: 65536
```

### Config YAML

The config values are dynamic based on the YAML provided, under the six top level
config options. Any of these can be ommitted if not used.

* fluentd_inputs: [{}] # creates `<source>`
* fluentd_ouputs: [{}] # creates `<match>`
* fluentd_filters: [{}] # creates `<filter>`
* fluentd_system: {} # creates `<system>`
* fluentd_labels: [{}] # creates `<label>`
* fluentd_includes: [""] # creates `@include`

There are some special keys `_pattern`, `_props`, and `_of`

* `_pattern` is added to the directive like `<filter [_pattern]>`.
* `_props` is used if order matters for any directive. 
* `_of` is used in the labels to specify the directive inside the label.
* `_ANYTHING` will be ignored.
* `true`, any value of true will just output the key name.

Any directive that has sub directives can be specified with a map of values 
for the sub directive. Use the directive for the name.

```yaml
fluentd_inputs:
  - type: 'tail'
    format: 'json'
    tag: 'my_app'
    path: '/mnt/my_app/log/production.log'
    pos_file: '/var/log/td-agent/my_app.log.pos'
    directive1:                       # Sub directive with a list of maps
      - d-thing: 1
        name: 'd1 t'
      - d-thing: 2
        name: 'd1 t'
    directive3:                       # Sub directive with a map of values
      d3-thing: 1
      name: 'd3-t'
  - type: 'other-ordered props'
    _pattern: '**'
    _props:                           # Order matters so using props
      - format: 'json'
      - tag: 'my_app'
      - path: '/mnt/my_app/log/production.log'
      - pos_file: '/var/log/td-agent/my_app.log.pos'
      - first: 'needs-first'
      - last: 'needs-last'

fluentd_ouputs:
  - type: "elasticsearch"
    port: 32422

fluentd_filters:
  - _pattern: 'foo.bar'
    type: parser
    format: '/^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)$/'
    time_format: '%d/%b/%Y:%H:%M:%S %z'
    key_name: message

  - _pattern: 'foo.bar'
    type: record_transformer
    enable_ruby: true
    record:
      avg: '${record["total"] / record["count"]}'

fluentd_system:
  log_level: error
  without_source: true
  process_name: fluentd1

fluentd_labels:
  - name: '@SYSTEM'
    directives: 
      - _of: 'filter'
        _pattern: 'var.log.middleware.**'
        type: 'grep'
      - _of: 'match'
        _pattern: '**'
        type: 's3'
        directive4:
          d4-thing: 1
          name: 'd4-t'

fluentd_includes:
  - "./config.d/*.config"
  - "http://example.com/fluent.conf"
```

## Dependencies

None

## Example Playbook

```yaml
---
- hosts: fluentd
  vars:
    fluentd_inputs:
      - type: 'tail'
        format: 'apache2'
        tag: 'apache.access'
        path: '/var/log/apache2/access.log'
      - type: 'tail'
        format: '/^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\] \[pid (?<pid>[^\]]*)\] \[client (?<client>[^\]]*)\] (?<message>.*)$/'
        tag: 'apache.error'
        path: '/var/log/apache2/error.log'
      - type: 'syslog'
        port: 5140
        bind: 0.0.0.0
        tag: 'syslog'
        protocol_type: 'udp'
      - type: 'tail'
        path: '/var/log/syslog'
        pos_file: '/var/log/syslog.pos'
        tag: 'syslog'
        format: 'syslog'
    fluentd_ouputs:
      - type: 'elasticsearch'
        logstash_format: true
        match: '**'
        host: '192.168.250.10'
        port: 9200
        index_name: 'fluentd'
        type_name: 'fluentd'
    fluentd_plugins:
      - 'fluent-plugin-elasticsearch'
    pri_domain_name: 'test.vagrant.local'
  roles:
    - role: ansible-fluentd
```

## License

BSD

## Author Information

Larry Smith Jr.

-   [@mrlesmithjr]
-   <http://everythingshouldbevirtual.com>
-   mrlesmithjr [at] gmail.com

[@mrlesmithjr]: https://www.twitter.com/mrlesmithjr

[ansible]: https://www.ansible.com

[fluentd]: http://www.fluentd.org/
