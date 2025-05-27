# #caladan

# Summary

`caladan` is a debian 12 VM running docker compose with a few services, including grafana, prometheus, and cyberchef. This VM is intended to run any internal management related stuff, or self hosted tools.  

Hardware

| memory     | 2 GB         |
| ---------- | ------------ |
| storage    | 32 GB        |
| processors | 1            |
| ip         | 10.0.0.13/24 |
| bridge     | vmbr0        |
| vlan       | 1            |

# Install and Setup

Setting this one up as a VM instead of a container, because from what I have heard it works better with docker. Not researching the intricacies right now, just trusting internet strangers. 

Install debian 12 without a desktop environment. I opted for LVM during the drive partitioning in case I need to resize later for logging. 

- **Grafana** to visualize logs.
  
- **Loki** to store logs, with chunks and indexes over NFS to the synology
  
- **Vector** to collect logs from the system and optionally Docker, then send them to Loki.

There will be a few config files needed for this setup. This folder schema is not required, but is what i am using: 

```sh
 mnt/ # NFS mount to log store on the Synology
 └── logs/
     ├── loki
     └── vector

 srv/
 └──caladan/
	 └── config/ # Configs for the services 
	     ├── loki
	     └── vector
```

## Docker Setup

Firstly, install docker per [[Host Setup#Docker]]. The 3 services above will be included as containers on this vm, using the following compose file: 

```yaml
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/loki-config.yaml
    volumes: 
      - ./config/loki/loki-config.yaml:/etc/loki/loki-config.yaml # Config File
      - /mnt/logs/loki:/loki # Log storage
    networks:
      - monitoring # Needed this due to errors

  grafana: # Basically copied from grafana docs 
    environment:
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_FEATURE_TOGGLES_ENABLE=alertingSimplifiedRouting,alertingQueryAndExpressionsStepMode
    entrypoint:
      - sh
      - -euc
      - |
        mkdir -p /etc/grafana/provisioning/datasources
        cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          access: proxy
          orgId: 1
          url: http://loki:3100
          basicAuth: false
          isDefault: true
          version: 1
          editable: false
        EOF
        /run.sh
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    restart: unless-stopped
    networks:
      - monitoring

  vector:
    image: timberio/vector:0.46.1-debian # No latest tag for vector
    container_name: vector
    volumes:
      - ./config/vector/vector-config.yaml:/etc/vector/vector.yaml # Vector Config
      - /var/log:/var/log:ro # Host Logs
      - /var/lib/docker/containers:/var/lib/docker/containers:ro # Docker Logs
      - /mnt/logs/vector:/vector-data # Vector Logs
    depends_on:
      - loki
    restart: unless-stopped
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge
```

Run this using the norm, although best to complete the rest of the config first: 

```sh
sudo docker compose pull && sudo docker compose up -d
```

Once running, you should be able to access grafana at `http://<host_ip>:3100`

***
## Loki Configuration

Loki is the log aggregator, and the config is stored in a yaml file. 

```yaml
auth_enabled: false

server:
  http_listen_port: 3100 # this is for loki endpoint
  grpc_listen_port: 9096 # This is for alert manager
  log_level: debug
  grpc_server_max_concurrent_streams: 1000

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki # Change this to your configured /loki log store location
  storage:
    filesystem:
      chunks_directory: /loki/chunks # Change these too to match above
      rules_directory: /loki/rules # Change these too to match above
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

limits_config:
  metric_aggregation_enabled: true

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

frontend:
  encoding: protobuf

# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/analytics/stats.go
# Refer to the buildReport method to see what goes into a report.
#
# If you would like to disable reporting, uncomment the following lines:
analytics:
  reporting_enabled: false
```

## Vector Configuration

Vector is the log collection agent that is running on each endpoint to collect logs and send to `loki`. There were a few options I considered: 

- syslog - if it aint broke, dont fix it
- promtail - comes with loki, easy setup
- vector - bit more advanced, more robust

Vector is a good choice since it is vendor agnostic, and designed for speed. The "transforms" for log parsing are also pretty useful from what I can tell, although a bit annoying to manage on each endpoint individually. Potentially, this is something I can just brute force later by creating a file with ALL types of logs, or might be able to just use some sort of aggregator for the transforms.   

Vector config is stored in `toml` or `yaml` files. See below config for `caladan`. It is pretty long, will comment with explanations

```yaml
sources:
  system_logs: # This is not really necessary for Debian
    type: file # Everything is stored in the Journal instead of syslog. 
    include:
      - /var/log/syslog
      - /var/log/auth.log
      - /var/log/kern.log

  docker_logs:
    type: file
    include:
      - /var/lib/docker/containers/*/*.log

  # The journald source requires you to include unit names of services to 
  # include. See below for some of the ones i chose
  journal:
    type: journald
    include_units: 
      - cron.service
      - ssh.service
      - networking.service
      - docker.service 
      - logind.service # Login activity
      - rpcbind.service # NFS Services
      - statd.service # NFS Services
      - gssd.service # NFS Services
    journal_directory: /var/log/journal

  auth_logs: # Empty so far, need to test
    type: file
    include:
      - /var/log/faillog/*/*.log
  
transforms:
  parse_docker_logs: # Docker log parsing
    type: remap
    inputs:
      - docker_logs
    source: |
      structured, err = parse_json(.message)
      if err == null {
        . = merge!(., structured)
      } else {
        . = . 
      }
      # .timestamp = parse_timestamp!(.time, "%+")  # Convert the time field to a proper timestamp
      .level = parse_json!(.message).log.level  # Extract the 'level' from the 'log' field
      .msg = parse_json!(.message).log.msg  # Extract the 'msg' from the 'log' field
      .stream = .message.stream  # Keep the stream info (stdout or stderr)
  parse_journal:
    type: remap
    inputs:
      - journal  # Name of your source, e.g., "docker_logs"
    source: |
      .timestamp = parse_timestamp!(.timestamp, "%+")  # Use actual timestamp field
    #  .level = parse_syslog!(.message).log.level  # Only if .message is plain syslog
    #  .msg = parse_syslog!(.message).log.msg
    

sinks:
  loki:
    type: loki
    inputs:
      - parse_docker_logs # Send these to loki
      - parse_journal # Send these to loki
    compression: snappy
    encoding:
      codec: json # Pick the right codec for your logs
    endpoint: http://loki:3100 
    request:
      in_flight_limit: 100  # Limit concurrent HTTP requests
      rate_limit_num: 50  # Enabled these to stop spamming the server
    labels:
      source: "vector"
      file: "{{ file }}"
      host: "{{ host }}"
      source_type: "{{ source_type }}"
    tenant_id: some_tenant_id

```

After all that, you can upload the file to your host, restart the `vector` container, and hopefully you will see logs flowing into grafana. 

***

Vector Agent Installation

To install a vector agent on a host, you can use compose: 

```yaml
  vector:
    image: timberio/vector:0.46.1-debian # No latest tag for vector
    container_name: vector
    volumes:
      - ./config/vector/vector-config.yaml:/etc/vector/vector.yaml # Vector Config
      - /var/log:/var/log:ro # Host Logs
      - /var/lib/docker/containers:/var/lib/docker/containers:ro # Docker Logs
      - /mnt/logs/vector:/vector-data # Vector Logs
    depends_on:
      - loki
    restart: unless-stopped
    networks:
      - monitoring
```

Configuring it is also done with a yaml file, see below: 

**NOTE!! - Lots of active work going on with ansible, etc. in my environment. As such, this doc is outdated and will be updated soon to include details of CI/CD workflows and other automations being worked on. 

https://vector.dev/docs/reference/configuration/sources/

```yaml
# Define your sources
sources:
  docker_logs:
    type: docker_logs

  journal:
    type: journald
    include_units:
      - cron.service
      - ssh.service
      - networking.service
      - docker.service
      - logind.service # Login activity
      - rpcbind.service # NFS Services
      - statd.service # NFS Services
      - gssd.service # NFS Services
    journal_directory: /var/log/journal

transforms:
  parse_docker_logs:
    type: remap
    inputs:
      - docker_logs  # Name of your source, e.g., "docker_logs"
    source: |
      parsed, err = parse_logfmt(.message)  # Parses message field, and outputs into event if no error. 
      if err == null {
        . = merge(., parsed)
      } else {
        log("Failed to parse syslog: " + err)
      }
    # .container_name = .container_name
    
    # .timestamp = parse_timestamp!(.ts, "%+")  # Convert the time field to a proper timestamp

    # del(.message)
    
  parse_journal:
    type: remap
    inputs:
      - journal  # Name of your source, e.g., "docker_logs"
    source: |
      parsed, err = parse_logfmt(.message)
      if err == null {
        . = merge(., parsed)
      } else {
        log("Failed to parse syslog: " + err)
      }

sinks:
  loki:
    type: loki
    inputs:
      - parse_docker_logs
      - parse_journal
    compression: snappy
    encoding:
      codec: json
    endpoint: http://loki:3100
    request:
      in_flight_limit: 100  # Limit concurrent HTTP requests
      rate_limit_num: 50  
    labels:
      source: "vector"
      container: "{{ container_name }}"
      host: "{{ host }}"
      source_type: "{{ source_type }}"
    tenant_id: some_tenant_id
```
