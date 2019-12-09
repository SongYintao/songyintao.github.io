---
title: Prometheus 文档
subtitle: prometheus 文档
layout: post
tags: [prometheus,metric]
---

 prometheus研究

[Prometheus学习文档](https://yunlzheng.gitbook.io/prometheus-book/)

[Prometheus+Kubernetes 部署参考](https://blog.csdn.net/liukuan73/article/details/78881008#)

[Webhook对接](https://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/)

# Configuration

​		Prometheus配置研究学习。

[Prometheus Config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

Prometheus is configured via **command-line flags** and **a configuration file**. While the command-line flags configure immutable system parameters (such as storage locations, amount of data to keep on disk and in memory, etc.), the configuration file defines everything related to scraping [jobs and their instances](https://prometheus.io/docs/concepts/jobs_instances/), as well as which [rule files to load](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#configuring-rules).

To view all available command-line flags, run `./prometheus -h`.

**Prometheus can reload its configuration at runtime**. If the new configuration is not well-formed, the changes will not be applied. A configuration reload is triggered by sending a `SIGHUP` to the Prometheus process or sending a HTTP POST request to the `/-/reload` endpoint (when the `--web.enable-lifecycle` flag is enabled). This will also reload any configured rule files.

## Configuration file

To specify which configuration file to load, use the `--config.file`flag.

The file is written in [YAML format](https://en.wikipedia.org/wiki/YAML), defined by the scheme described below. Brackets indicate that a parameter is optional. For non-list parameters the value is set to the specified default.

Generic placeholders are defined as follows:

- `<boolean>`: a boolean that can take the values `true` or `false`
- `<duration>`: a duration matching the regular expression `[0-9]+(ms|[smhdwy])`
- `<labelname>`: a string matching the regular expression `[a-zA-Z_][a-zA-Z0-9_]*`
- `<labelvalue>`: a string of unicode characters
- `<filename>`: a valid path in the current working directory
- `<host>`: a valid string consisting of a hostname or IP followed by an optional port number
- `<path>`: a valid URL path
- `<scheme>`: a string that can take the values `http` or `https`
- `<string>`: a regular string
- `<secret>`: a regular string that is a secret, such as a password
- `<tmpl_string>`: a string which is template-expanded before usage

The other placeholders are specified separately.

A valid example file can be found [here](https://github.com/prometheus/prometheus/blob/release-2.9/config/testdata/conf.good.yml).

The global configuration specifies parameters that are valid in all other configuration contexts. They also serve as defaults for other configuration sections.

```yaml
global:
  # How frequently to scrape targets by default.
  [ scrape_interval: <duration> | default = 1m ]

  # How long until a scrape request times out.
  [ scrape_timeout: <duration> | default = 10s ]

  # How frequently to evaluate rules.
  [ evaluation_interval: <duration> | default = 1m ]

  # The labels to add to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    [ <labelname>: <labelvalue> ... ]

# Rule files specifies a list of globs. Rules and alerts are read from
# all matching files. 
# 规则文件 指定了一个globs列表。 
rule_files:
  [ - <filepath_glob> ... ]

# A list of scrape configurations. 
scrape_configs:
  [ - <scrape_config> ... ]

# Alerting specifies settings related to the Alertmanager.
# Alertmanager 相关配置
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]

# Settings related to the remote write feature.
remote_write:
  [ - <remote_write> ... ]

# Settings related to the remote read feature.
remote_read:
  [ - <remote_read> ... ]
```

### `<scrape_config>`   scrape配置

A `scrape_config` section specifies a set of targets and parameters describing how to scrape them. **In the general case, one scrape configuration specifies a single job**. In advanced configurations, this may change.

**一个scrape 配置对应一个Job**。

Targets may be statically configured via the `static_configs` parameter or dynamically discovered using one of the supported service-discovery mechanisms.

Additionally, `relabel_configs` allow advanced modifications to any target and its labels before scraping.

```yaml
# The job name assigned to scraped metrics by default.
job_name: <job_name>

# How frequently to scrape targets from this job.
[ scrape_interval: <duration> | default = <global_config.scrape_interval> ]

# Per-scrape timeout when scraping this job.
[ scrape_timeout: <duration> | default = <global_config.scrape_timeout> ]

# The HTTP resource path on which to fetch metrics from targets.
#Http metric的路径，用于targets参数，提取metric参数信息。
[ metrics_path: <path> | default = /metrics ]

# honor_labels controls how Prometheus handles conflicts between labels that are
# already present in scraped data and labels that Prometheus would attach
# server-side ("job" and "instance" labels, manually configured target
# labels, and labels generated by service discovery implementations).
#
# If honor_labels is set to "true", label conflicts are resolved by keeping label
# values from the scraped data and ignoring the conflicting server-side labels.

#honor_labels 为true，标签冲突时，忽略server端标签。

# If honor_labels is set to "false", label conflicts are resolved by renaming
# conflicting labels in the scraped data to "exported_<original-label>" (for
# example "exported_instance", "exported_job") and then attaching server-side
# labels. This is useful for use cases such as federation, where all labels
# specified in the target should be preserved.

#honor_labels 为true，添加 external_labels

# Note that any globally configured "external_labels" are unaffected by this
# setting. In communication with external systems, they are always applied only
# when a time series does not have a given label yet and are ignored otherwise.
[ honor_labels: <boolean> | default = false ]

# honor_timestamps controls whether Prometheus respects the timestamps present
# in scraped data.
#
# If honor_timestamps is set to "true", the timestamps of the metrics exposed
# by the target will be used.
#
# If honor_timestamps is set to "false", the timestamps of the metrics exposed
# by the target will be ignored.
[ honor_timestamps: <boolean> | default = true ]

# Configures the protocol scheme used for requests.
[ scheme: <scheme> | default = http ]

# Optional HTTP URL parameters.  url参数
params:
  [ <string>: [<string>, ...] ]

# Sets the `Authorization` header on every scrape request with the
# configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every scrape request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file`.
# 持有的token
[ bearer_token: <secret> ]

# Sets the `Authorization` header on every scrape request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the scrape request's TLS settings.
#请求相关的tls配置
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# List of Azure service discovery configurations.
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# List of Consul service discovery configurations.
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# List of DNS service discovery configurations.
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# List of EC2 service discovery configurations.
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# List of OpenStack service discovery configurations.
openstack_sd_configs:
  [ - <openstack_sd_config> ... ]

# List of file service discovery configurations.
file_sd_configs:
  [ - <file_sd_config> ... ]

# List of GCE service discovery configurations.
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# List of Kubernetes service discovery configurations.
# kubernetes服务发现的配置
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# List of Marathon service discovery configurations.
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# List of AirBnB's Nerve service discovery configurations.
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# List of Zookeeper Serverset service discovery configurations.
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# List of Triton service discovery configurations.
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# List of labeled statically configured targets for this job.
static_configs:
  [ - <static_config> ... ]

# List of target relabel configurations.
# target重新打标签配置
relabel_configs:
  [ - <relabel_config> ... ]

# List of metric relabel configurations.
# metric重新打标签配置
metric_relabel_configs:
  [ - <relabel_config> ... ]

# Per-scrape limit on number of scraped samples that will be accepted.
# If more than this number of samples are present after metric relabelling
# the entire scrape will be treated as failed. 0 means no limit.
[ sample_limit: <int> | default = 0 ]
```

Where `<job_name>` must be unique across all scrape configurations.

### `<tls_config>` TLS配置

A `tls_config` allows configuring TLS connections.  配置相关的TLS连接。

```yaml
# CA certificate to validate API server certificate with.
# 用于校验API的服务端证书
[ ca_file: <filename> ]

# Certificate and key files for client cert authentication to the server.
#客户端用于连接服务器的证书和私钥文件
[ cert_file: <filename> ]
[ key_file: <filename> ]

# ServerName extension to indicate the name of the server.
# https://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# Disable validation of the server certificate.
[ insecure_skip_verify: <boolean> ]
```

### `<azure_sd_config>`

Azure SD configurations allow retrieving scrape targets from Azure VMs.

The following meta labels are available on targets during relabeling:

- `__meta_azure_machine_id`: the machine ID
- `__meta_azure_machine_location`: the location the machine runs in
- `__meta_azure_machine_name`: the machine name
- `__meta_azure_machine_os_type`: the machine operating system
- `__meta_azure_machine_private_ip`: the machine's private IP
- `__meta_azure_machine_resource_group`: the machine's resource group
- `__meta_azure_machine_tag_<tagname>`: each tag value of the machine
- `__meta_azure_machine_scale_set`: the name of the scale set which the vm is part of (this value is only set if you are using a [scale set](https://docs.microsoft.com/en-us/azure/virtual-machine-scale-sets/))
- `__meta_azure_subscription_id`: the subscription ID
- `__meta_azure_tenant_id`: the tenant ID

See below for the configuration options for Azure discovery:

```yaml
# The information to access the Azure API.
# The Azure environment.
[ environment: <string> | default = AzurePublicCloud ]

# The authentication method, either OAuth or ManagedIdentity.
# See https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview
[ authentication_method: <string> | default = OAuth]
# The subscription ID. Always required.
subscription_id: <string>
# Optional tenant ID. Only required with authentication_method OAuth.
[ tenant_id: <string> ]
# Optional client ID. Only required with authentication_method OAuth.
[ client_id: <string> ]
# Optional client secret. Only required with authentication_method OAuth.
[ client_secret: <secret> ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 300s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]
```

### `<consul_sd_config>`

Consul SD configurations allow retrieving scrape targets from [Consul's](https://www.consul.io/) Catalog API.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_consul_address`: the address of the target
- `__meta_consul_dc`: the datacenter name for the target
- `__meta_consul_tagged_address_<key>`: each node tagged address key value of the target
- `__meta_consul_metadata_<key>`: each node metadata key value of the target
- `__meta_consul_node`: the node name defined for the target
- `__meta_consul_service_address`: the service address of the target
- `__meta_consul_service_id`: the service ID of the target
- `__meta_consul_service_metadata_<key>`: each service metadata key value of the target
- `__meta_consul_service_port`: the service port of the target
- `__meta_consul_service`: the name of the service the target belongs to
- `__meta_consul_tags`: the list of tags of the target joined by the tag separator

```yaml
# The information to access the Consul API. It is to be defined
# as the Consul documentation requires.
[ server: <host> | default = "localhost:8500" ]
[ token: <secret> ]
[ datacenter: <string> ]
[ scheme: <string> | default = "http" ]
[ username: <string> ]
[ password: <secret> ]

tls_config:
  [ <tls_config> ]

# A list of services for which targets are retrieved. If omitted, all services
# are scraped.
services:
  [ - <string> ]

# See https://www.consul.io/api/catalog.html#list-nodes-for-service to know more
# about the possible filters that can be used.

# An optional list of tags used to filter nodes for a given service. Services must contain all tags in the list.
tags:
  [ - <string> ]

# Node metadata used to filter nodes for a given service.
[ node_meta:
  [ <name>: <value> ... ] ]

# The string by which Consul tags are joined into the tag label.
[ tag_separator: <string> | default = , ]

# Allow stale Consul results (see https://www.consul.io/api/index.html#consistency-modes). Will reduce load on Consul.
[ allow_stale: <bool> ]

# The time after which the provided names are refreshed.
# On large setup it might be a good idea to increase this value because the catalog will change all the time.
[ refresh_interval: <duration> | default = 30s ]
```

Note that the IP number and port used to scrape the targets is assembled as `<__meta_consul_address>:<__meta_consul_service_port>`. However, in some Consul setups, the relevant address is in `__meta_consul_service_address`. In those cases, you can use the [relabel](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) feature to replace the special `__address__` label.

The [relabeling phase](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) is the preferred and more powerful way to filter services or nodes for a service based on arbitrary labels. For users with thousands of services it can be more efficient to use the Consul API directly which has basic support for filtering nodes (currently by node metadata and a single tag).

### `<dns_sd_config>`

A DNS-based service discovery configuration allows specifying a set of DNS domain names which are periodically queried to discover a list of targets. The DNS servers to be contacted are read from `/etc/resolv.conf`.

This service discovery method only supports basic DNS A, AAAA and SRV record queries, but not the advanced DNS-SD approach specified in [RFC6763](https://tools.ietf.org/html/rfc6763).

During the [relabeling phase](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config), the meta label `__meta_dns_name` is available on each target and is set to the record name that produced the discovered target.

```yaml
# A list of DNS domain names to be queried.
names:
  [ - <domain_name> ]

# The type of DNS query to perform.
[ type: <query_type> | default = 'SRV' ]

# The port number used if the query type is not SRV.
[ port: <number>]

# The time after which the provided names are refreshed.
[ refresh_interval: <duration> | default = 30s ]
```

Where `<domain_name>` is a valid DNS domain name. Where `<query_type>` is `SRV`, `A`, or `AAAA`.

### `<ec2_sd_config>`

EC2 SD configurations allow retrieving scrape targets from AWS EC2 instances. The private IP address is used by default, but may be changed to the public IP address with relabeling.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_ec2_availability_zone`: the availability zone in which the instance is running
- `__meta_ec2_instance_id`: the EC2 instance ID
- `__meta_ec2_instance_state`: the state of the EC2 instance
- `__meta_ec2_instance_type`: the type of the EC2 instance
- `__meta_ec2_owner_id`: the ID of the AWS account that owns the EC2 instance
- `__meta_ec2_platform`: the Operating System platform, set to 'windows' on Windows servers, absent otherwise
- `__meta_ec2_primary_subnet_id`: the subnet ID of the primary network interface, if available
- `__meta_ec2_private_dns_name`: the private DNS name of the instance, if available
- `__meta_ec2_private_ip`: the private IP address of the instance, if present
- `__meta_ec2_public_dns_name`: the public DNS name of the instance, if available
- `__meta_ec2_public_ip`: the public IP address of the instance, if available
- `__meta_ec2_subnet_id`: comma separated list of subnets IDs in which the instance is running, if available
- `__meta_ec2_tag_<tagkey>`: each tag value of the instance
- `__meta_ec2_vpc_id`: the ID of the VPC in which the instance is running, if available

See below for the configuration options for EC2 discovery:

```yaml
# The information to access the EC2 API.

# The AWS Region.
region: <string>

# Custom endpoint to be used.
[ endpoint: <string> ]

# The AWS API keys. If blank, the environment variables `AWS_ACCESS_KEY_ID`
# and `AWS_SECRET_ACCESS_KEY` are used.
[ access_key: <string> ]
[ secret_key: <secret> ]
# Named AWS profile used to connect to the API.
[ profile: <string> ]

# AWS Role ARN, an alternative to using AWS API keys.
[ role_arn: <string> ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# Filters can be used optionally to filter the instance list by other criteria.
# Available filter criteria can be found here:
# https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_DescribeInstances.html
# Filter API documentation: https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_Filter.html
filters:
  [ - name: <string>
      values: <string>, [...] ]
```

The [relabeling phase](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config) is the preferred and more powerful way to filter targets based on arbitrary labels. For users with thousands of instances it can be more efficient to use the EC2 API directly which has support for filtering instances.

### `<openstack_sd_config>`

OpenStack SD configurations allow retrieving scrape targets from OpenStack Nova instances.

One of the following `<openstack_role>` types can be configured to discover targets:

#### `hypervisor`

The `hypervisor` role discovers one target per Nova hypervisor node. The target address defaults to the `host_ip`attribute of the hypervisor.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_openstack_hypervisor_host_ip`: the hypervisor node's IP address.
- `__meta_openstack_hypervisor_name`: the hypervisor node's name.
- `__meta_openstack_hypervisor_state`: the hypervisor node's state.
- `__meta_openstack_hypervisor_status`: the hypervisor node's status.
- `__meta_openstack_hypervisor_type`: the hypervisor node's type.

#### `instance`

The `instance` role discovers one target per network interface of Nova instance. The target address defaults to the private IP address of the network interface.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_openstack_address_pool`: the pool of the private IP.
- `__meta_openstack_instance_flavor`: the flavor of the OpenStack instance.
- `__meta_openstack_instance_id`: the OpenStack instance ID.
- `__meta_openstack_instance_name`: the OpenStack instance name.
- `__meta_openstack_instance_status`: the status of the OpenStack instance.
- `__meta_openstack_private_ip`: the private IP of the OpenStack instance.
- `__meta_openstack_project_id`: the project (tenant) owning this instance.
- `__meta_openstack_public_ip`: the public IP of the OpenStack instance.
- `__meta_openstack_tag_<tagkey>`: each tag value of the instance.
- `__meta_openstack_user_id`: the user account owning the tenant.

See below for the configuration options for OpenStack discovery:

```yaml
# The information to access the OpenStack API.

# The OpenStack role of entities that should be discovered.
role: <openstack_role>

# The OpenStack Region.
region: <string>

# identity_endpoint specifies the HTTP endpoint that is required to work with
# the Identity API of the appropriate version. While it's ultimately needed by
# all of the identity services, it will often be populated by a provider-level
# function.
[ identity_endpoint: <string> ]

# username is required if using Identity V2 API. Consult with your provider's
# control panel to discover your account's username. In Identity V3, either
# userid or a combination of username and domain_id or domain_name are needed.
[ username: <string> ]
[ userid: <string> ]

# password for the Identity V2 and V3 APIs. Consult with your provider's
# control panel to discover your account's preferred method of authentication.
[ password: <secret> ]

# At most one of domain_id and domain_name must be provided if using username
# with Identity V3. Otherwise, either are optional.
[ domain_name: <string> ]
[ domain_id: <string> ]

# The project_id and project_name fields are optional for the Identity V2 API.
# Some providers allow you to specify a project_name instead of the project_id.
# Some require both. Your provider's authentication policies will determine
# how these fields influence authentication.
[ project_name: <string> ]
[ project_id: <string> ]

# The application_credential_id or application_credential_name fields are
# required if using an application credential to authenticate. Some providers
# allow you to create an application credential to authenticate rather than a
# password.
[ application_credential_name: <string> ]
[ application_credential_id: <string> ]

# The application_credential_secret field is required if using an application
# credential to authenticate.
[ application_credential_secret: <secret> ]

# Whether the service discovery should list all instances for all projects.
# It is only relevant for the 'instance' role and usually requires admin permissions.
[ all_tenants: <boolean> | default: false ]

# Refresh interval to re-read the instance list.
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# TLS configuration.
tls_config:
  [ <tls_config> ]
```

### `<file_sd_config>`

File-based service discovery provides a more generic way to configure static targets and serves as an interface to plug in custom service discovery mechanisms.

It reads a set of files containing a list of zero or more `<static_config>`s. Changes to all defined files are detected via disk watches and applied immediately. Files may be provided in YAML or JSON format. Only changes resulting in well-formed target groups are applied.

The JSON file must contain a list of static configs, using this format:

```yaml
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

As a fallback, the file contents are also re-read periodically at the specified refresh interval.

Each target has a meta label `__meta_filepath` during the [relabeling phase](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config). Its value is set to the filepath from which the target was extracted.

There is a list of [integrations](https://prometheus.io/docs/operating/integrations/#file-service-discovery) with this discovery mechanism.

```yaml
# Patterns for files from which target groups are extracted.
files:
  [ - <filename_pattern> ... ]

# Refresh interval to re-read the files.
[ refresh_interval: <duration> | default = 5m ]
```

Where `<filename_pattern>` may be a path ending in `.json`, `.yml` or `.yaml`. The last path segment may contain a single `*` that matches any character sequence, e.g. `my/path/tg_*.json`.

### `<gce_sd_config>`

[GCE](https://cloud.google.com/compute/) SD configurations allow retrieving scrape targets from GCP GCE instances. The private IP address is used by default, but may be changed to the public IP address with relabeling.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_gce_instance_id`: the numeric id of the instance
- `__meta_gce_instance_name`: the name of the instance
- `__meta_gce_label_<name>`: each GCE label of the instance
- `__meta_gce_machine_type`: full or partial URL of the machine type of the instance
- `__meta_gce_metadata_<name>`: each metadata item of the instance
- `__meta_gce_network`: the network URL of the instance
- `__meta_gce_private_ip`: the private IP address of the instance
- `__meta_gce_project`: the GCP project in which the instance is running
- `__meta_gce_public_ip`: the public IP address of the instance, if present
- `__meta_gce_subnetwork`: the subnetwork URL of the instance
- `__meta_gce_tags`: comma separated list of instance tags
- `__meta_gce_zone`: the GCE zone URL in which the instance is running

See below for the configuration options for GCE discovery:

```yaml
# The information to access the GCE API.

# The GCP Project
project: <string>

# The zone of the scrape targets. If you need multiple zones use multiple
# gce_sd_configs.
zone: <string>

# Filter can be used optionally to filter the instance list by other criteria
# Syntax of this filter string is described here in the filter query parameter section:
# https://cloud.google.com/compute/docs/reference/latest/instances/list
[ filter: <string> ]

# Refresh interval to re-read the instance list
[ refresh_interval: <duration> | default = 60s ]

# The port to scrape metrics from. If using the public IP address, this must
# instead be specified in the relabeling rule.
[ port: <int> | default = 80 ]

# The tag separator is used to separate the tags on concatenation
[ tag_separator: <string> | default = , ]
```

Credentials are discovered by the Google Cloud SDK default client by looking in the following places, preferring the first location found:

1. a JSON file specified by the `GOOGLE_APPLICATION_CREDENTIALS` environment variable
2. a JSON file in the well-known path `$HOME/.config/gcloud/application_default_credentials.json`
3. fetched from the GCE metadata server

If Prometheus is running within GCE, the service account associated with the instance it is running on should have at least read-only permissions to the compute resources. If running outside of GCE make sure to create an appropriate service account and place the credential file in one of the expected locations.

### `<kubernetes_sd_config>`

Kubernetes SD configurations allow retrieving scrape targets from [Kubernetes'](https://kubernetes.io/) REST API and always staying synchronized with the cluster state.

One of the following `role` types can be configured to discover targets:

#### `node`

The `node` role discovers one target per cluster node with the address defaulting to the Kubelet's HTTP port. The target address defaults to the first existing address of the Kubernetes node object in the address type order of `NodeInternalIP`, `NodeExternalIP`, `NodeLegacyHostIP`, and `NodeHostName`.

Available meta labels:

- `__meta_kubernetes_node_name`: The name of the node object.
- `__meta_kubernetes_node_label_<labelname>`: Each label from the node object.
- `__meta_kubernetes_node_labelpresent_<labelname>`: `true` for each label from the node object.
- `__meta_kubernetes_node_annotation_<annotationname>`: Each annotation from the node object.
- `__meta_kubernetes_node_annotationpresent_<annotationname>`: `true` for each annotation from the node object.
- `__meta_kubernetes_node_address_<address_type>`: The first address for each node address type, if it exists.

In addition, the `instance` label for the node will be set to the node name as retrieved from the API server.

#### `service`

The `service` role discovers a target for each service port for each service. This is generally useful for blackbox monitoring of a service. The address will be set to the Kubernetes DNS name of the service and respective service port.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the service object.
- `__meta_kubernetes_service_annotation_<annotationname>`: Each annotation from the service object.
- `__meta_kubernetes_service_annotationpresent_<annotationname>`: "true" for each annotation of the service object.
- `__meta_kubernetes_service_cluster_ip`: The cluster IP address of the service. (Does not apply to services of type ExternalName)
- `__meta_kubernetes_service_external_name`: The DNS name of the service. (Applies to services of type ExternalName)
- `__meta_kubernetes_service_label_<labelname>`: Each label from the service object.
- `__meta_kubernetes_service_labelpresent_<labelname>`: `true` for each label of the service object.
- `__meta_kubernetes_service_name`: The name of the service object.
- `__meta_kubernetes_service_port_name`: Name of the service port for the target.
- `__meta_kubernetes_service_port_number`: Number of the service port for the target.
- `__meta_kubernetes_service_port_protocol`: Protocol of the service port for the target.

#### `pod`

The `pod` role discovers all pods and exposes their containers as targets. For each declared port of a container, a single target is generated. If a container has no specified ports, a port-free target per container is created for manually adding a port via relabeling.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the pod object.
- `__meta_kubernetes_pod_name`: The name of the pod object.
- `__meta_kubernetes_pod_ip`: The pod IP of the pod object.
- `__meta_kubernetes_pod_label_<labelname>`: Each label from the pod object.
- `__meta_kubernetes_pod_labelpresent_<labelname>`: `true`for each label from the pod object.
- `__meta_kubernetes_pod_annotation_<annotationname>`: Each annotation from the pod object.
- `__meta_kubernetes_pod_annotationpresent_<annotationname>`: `true` for each annotation from the pod object.
- `__meta_kubernetes_pod_container_name`: Name of the container the target address points to.
- `__meta_kubernetes_pod_container_port_name`: Name of the container port.
- `__meta_kubernetes_pod_container_port_number`: Number of the container port.
- `__meta_kubernetes_pod_container_port_protocol`: Protocol of the container port.
- `__meta_kubernetes_pod_ready`: Set to `true` or `false` for the pod's ready state.
- `__meta_kubernetes_pod_phase`: Set to `Pending`, `Running`, `Succeeded`, `Failed` or `Unknown` in the [lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase).
- `__meta_kubernetes_pod_node_name`: The name of the node the pod is scheduled onto.
- `__meta_kubernetes_pod_host_ip`: The current host IP of the pod object.
- `__meta_kubernetes_pod_uid`: The UID of the pod object.
- `__meta_kubernetes_pod_controller_kind`: Object kind of the pod controller.
- `__meta_kubernetes_pod_controller_name`: Name of the pod controller.

#### `endpoints`

The `endpoints` role discovers targets from listed endpoints of a service. For each endpoint address one target is discovered per port. If the endpoint is backed by a pod, all additional container ports of the pod, not bound to an endpoint port, are discovered as targets as well.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the endpoints object.
- `__meta_kubernetes_endpoints_name`: The names of the endpoints object.
- For all targets discovered directly from the endpoints list (those not additionally inferred from underlying pods), the following labels are attached:
  - `__meta_kubernetes_endpoint_ready`: Set to `true` or `false` for the endpoint's ready state.
  - `__meta_kubernetes_endpoint_port_name`: Name of the endpoint port.
  - `__meta_kubernetes_endpoint_port_protocol`: Protocol of the endpoint port.
  - `__meta_kubernetes_endpoint_address_target_kind`: Kind of the endpoint address target.
  - `__meta_kubernetes_endpoint_address_target_name`: Name of the endpoint address target.
- If the endpoints belong to a service, all labels of the `role: service` discovery are attached.
- For all targets backed by a pod, all labels of the `role: pod` discovery are attached.

#### `ingress`

The `ingress` role discovers a target for each path of each ingress. This is generally useful for blackbox monitoring of an ingress. The address will be set to the host specified in the ingress spec.

Available meta labels:

- `__meta_kubernetes_namespace`: The namespace of the ingress object.
- `__meta_kubernetes_ingress_name`: The name of the ingress object.
- `__meta_kubernetes_ingress_label_<labelname>`: Each label from the ingress object.
- `__meta_kubernetes_ingress_labelpresent_<labelname>`: `true` for each label from the ingress object.
- `__meta_kubernetes_ingress_annotation_<annotationname>`: Each annotation from the ingress object.
- `__meta_kubernetes_ingress_annotationpresent_<annotationname>`: `true` for each annotation from the ingress object.
- `__meta_kubernetes_ingress_scheme`: Protocol scheme of ingress, `https` if TLS config is set. Defaults to `http`.
- `__meta_kubernetes_ingress_path`: Path from ingress spec. Defaults to `/`.

See below for the configuration options for Kubernetes discovery:

```yaml
# The information to access the Kubernetes API.

# The API server addresses. If left empty, Prometheus is assumed to run inside
# of the cluster and will discover API servers automatically and use the pod's
# CA certificate and bearer token file at /var/run/secrets/kubernetes.io/serviceaccount/.
[ api_server: <host> ]

# The Kubernetes role of entities that should be discovered.
role: <role>

# Optional authentication information used to authenticate to the API server.
# Note that `basic_auth`, `bearer_token` and `bearer_token_file` options are
# mutually exclusive.
# password and password_file are mutually exclusive.

# Optional HTTP basic authentication information.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Optional bearer token authentication information.
[ bearer_token: <secret> ]

# Optional bearer token file authentication information.
[ bearer_token_file: <filename> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# TLS configuration.
tls_config:
  [ <tls_config> ]

# Optional namespace discovery. If omitted, all namespaces are used.
namespaces:
  names:
    [ - <string> ]
```

Where `<role>` must be `endpoints`, `service`, `pod`, `node`, or `ingress`.

See [this example Prometheus configuration file](https://github.com/prometheus/prometheus/blob/release-2.9/documentation/examples/prometheus-kubernetes.yml) for a detailed example of configuring Prometheus for Kubernetes.

You may wish to check out the 3rd party [Prometheus Operator](https://github.com/coreos/prometheus-operator), which automates the Prometheus setup on top of Kubernetes.

### `<marathon_sd_config>`

Marathon SD configurations allow retrieving scrape targets using the [Marathon](https://mesosphere.github.io/marathon/) REST API. Prometheus will periodically check the REST endpoint for currently running tasks and create a target group for every app that has at least one healthy task.

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_marathon_app`: the name of the app (with slashes replaced by dashes)
- `__meta_marathon_image`: the name of the Docker image used (if available)
- `__meta_marathon_task`: the ID of the Mesos task
- `__meta_marathon_app_label_<labelname>`: any Marathon labels attached to the app
- `__meta_marathon_port_definition_label_<labelname>`: the port definition labels
- `__meta_marathon_port_mapping_label_<labelname>`: the port mapping labels
- `__meta_marathon_port_index`: the port index number (e.g. `1` for `PORT1`)

See below for the configuration options for Marathon discovery:

```yaml
# List of URLs to be used to contact Marathon servers.
# You need to provide at least one server URL.
servers:
  - <string>

# Polling interval
[ refresh_interval: <duration> | default = 30s ]

# Optional authentication information for token-based authentication
# https://docs.mesosphere.com/1.11/security/ent/iam-api/#passing-an-authentication-token
# It is mutually exclusive with `auth_token_file` and other authentication mechanisms.
[ auth_token: <secret> ]

# Optional authentication information for token-based authentication
# https://docs.mesosphere.com/1.11/security/ent/iam-api/#passing-an-authentication-token
# It is mutually exclusive with `auth_token` and other authentication mechanisms.
[ auth_token_file: <filename> ]

# Sets the `Authorization` header on every request with the
# configured username and password.
# This is mutually exclusive with other authentication mechanisms.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file` and other authentication mechanisms.
# NOTE: The current version of DC/OS marathon (v1.11.0) does not support standard Bearer token authentication. Use `auth_token` instead.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token` and other authentication mechanisms.
# NOTE: The current version of DC/OS marathon (v1.11.0) does not support standard Bearer token authentication. Use `auth_token_file` instead.
[ bearer_token_file: /path/to/bearer/token/file ]

# TLS configuration for connecting to marathon servers
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```

By default every app listed in Marathon will be scraped by Prometheus. If not all of your services provide Prometheus metrics, you can use a Marathon label and Prometheus relabeling to control which instances will actually be scraped. See [the Prometheus marathon-sd configuration file](https://github.com/prometheus/prometheus/blob/release-2.9/documentation/examples/prometheus-marathon.yml) for a practical example on how to set up your Marathon app and your Prometheus configuration.

By default, all apps will show up as a single job in Prometheus (the one specified in the configuration file), which can also be changed using relabeling.

### `<nerve_sd_config>`

Nerve SD configurations allow retrieving scrape targets from [AirBnB's Nerve](https://github.com/airbnb/nerve) which are stored in [Zookeeper](https://zookeeper.apache.org/).

The following meta labels are available on targets during [relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config):

- `__meta_nerve_path`: the full path to the endpoint node in Zookeeper
- `__meta_nerve_endpoint_host`: the host of the endpoint
- `__meta_nerve_endpoint_port`: the port of the endpoint
- `__meta_nerve_endpoint_name`: the name of the endpoint

```yaml
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single service, or the root of a tree of services.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

### `<serverset_sd_config>`

Serverset SD configurations allow retrieving scrape targets from [Serversets](https://github.com/twitter/finagle/tree/master/finagle-serversets) which are stored in [Zookeeper](https://zookeeper.apache.org/). Serversets are commonly used by [Finagle](https://twitter.github.io/finagle/) and [Aurora](https://aurora.apache.org/).

The following meta labels are available on targets during relabeling:

- `__meta_serverset_path`: the full path to the serverset member node in Zookeeper
- `__meta_serverset_endpoint_host`: the host of the default endpoint
- `__meta_serverset_endpoint_port`: the port of the default endpoint
- `__meta_serverset_endpoint_host_<endpoint>`: the host of the given endpoint
- `__meta_serverset_endpoint_port_<endpoint>`: the port of the given endpoint
- `__meta_serverset_shard`: the shard number of the member
- `__meta_serverset_status`: the status of the member

```yaml
# The Zookeeper servers.
servers:
  - <host>
# Paths can point to a single serverset, or the root of a tree of serversets.
paths:
  - <string>
[ timeout: <duration> | default = 10s ]
```

Serverset data must be in the JSON format, the Thrift format is not currently supported.

### `<triton_sd_config>`

[Triton](https://github.com/joyent/triton) SD configurations allow retrieving scrape targets from [Container Monitor](https://github.com/joyent/rfd/blob/master/rfd/0027/README.md) discovery endpoints.

The following meta labels are available on targets during relabeling:

- `__meta_triton_groups`: the list of groups belonging to the target joined by a comma separator
- `__meta_triton_machine_alias`: the alias of the target container
- `__meta_triton_machine_brand`: the brand of the target container
- `__meta_triton_machine_id`: the UUID of the target container
- `__meta_triton_machine_image`: the target containers image type
- `__meta_triton_server_id`: the server UUID for the target container

```yaml
# The information to access the Triton discovery API.

# The account to use for discovering new target containers.
account: <string>

# The DNS suffix which should be applied to target containers.
dns_suffix: <string>

# The Triton discovery endpoint (e.g. 'cmon.us-east-3b.triton.zone'). This is
# often the same value as dns_suffix.
endpoint: <string>

# A list of groups for which targets are retrieved. If omitted, all containers
# available to the requesting account are scraped.
groups:
  [ - <string> ... ]

# The port to use for discovery and metric scraping.
[ port: <int> | default = 9163 ]

# The interval which should be used for refreshing target containers.
[ refresh_interval: <duration> | default = 60s ]

# The Triton discovery API version.
[ version: <int> | default = 1 ]

# TLS configuration.
tls_config:
  [ <tls_config> ]
```

### `<static_config>` 静态配置

A `static_config` allows specifying a list of targets and a common label set for them. It is the canonical way to specify static targets in a scrape configuration.

```yaml
# The targets specified by the static config.
targets:
  [ - '<host>' ]

# Labels assigned to all metrics scraped from the targets.
labels:
  [ <labelname>: <labelvalue> ... ]
```

### `<relabel_config>` 格式换标签相关配置

**Relabeling is a powerful tool to dynamically rewrite the label set of a target before it gets scraped.** Multiple relabeling steps can be configured per scrape configuration. They are applied to the label set of each target in order of their appearance in the configuration file.

Initially, aside from the configured per-target labels, a target's `job` label is set to the `job_name` value of the respective scrape configuration. The `__address__` label is set to the `<host>:<port>` address of the target. After relabeling, the `instance` label is set to the value of `__address__` by default if it was not set during relabeling. The `__scheme__` and `__metrics_path__` labels are set to the scheme and metrics path of the target respectively. The `__param_<name>` label is set to the value of the first passed URL parameter called `<name>`.

一个target的job标签设置到各自scrape配置的job_name值。` __addres__`标签设置为target的<host>:<port>地址。 在格式化标签之后， `instance`标签设置为`__address__`值。`__scheme__`和`__metrics_path__`标签设置为对应的配置值。 `__param_<name>` 标签设置为传递参数的URL参数`<name>`。

Additional labels **prefixed with `__meta_` may be available during the relabeling phase**. They are set by the service discovery mechanism that provided the target and vary between mechanisms.

Labels **starting with `__` will be removed from the label set after target relabeling is completed**.

临时参数：

**If a relabeling step needs to store a label value only temporarily (as the input to a subsequent relabeling step), use the `__tmp` label name prefix.** **This prefix is guaranteed to never be used by Prometheus itself.**

```yaml
# The source labels select values from existing labels. Their content is concatenated
# using the configured separator and matched against the configured regular expression
# for the replace, keep, and drop actions.
# 源标签：从当前存在的标签里面选择标签。他们的内容由下面的东西拼接而成：配置的分隔符、匹配的正则表达（用于代替、保持、丢弃）
[ source_labels: '[' <labelname> [, ...] ']' ]

# Separator placed between concatenated source label values.
# 源标签值之间的分割符
[ separator: <string> | default = ; ]

# Label to which the resulting value is written in a replace action.
# It is mandatory for replace actions. Regex capture groups are available.
# 在replace action里面的结果值，只适用于 replace操作
[ target_label: <labelname> ]

# Regular expression against which the extracted value is matched.
[ regex: <regex> | default = (.*) ]

# Modulus to take of the hash of the source label values.
[ modulus: <uint64> ]

# Replacement value against which a regex replace is performed if the
# regular expression matches. Regex capture groups are available.
[ replacement: <string> | default = $1 ]

# Action to perform based on regex matching.
[ action: <relabel_action> | default = replace ]
```

`<regex>` is any valid [RE2 regular expression](https://github.com/google/re2/wiki/Syntax). It is required for the `replace`, `keep`, `drop`, `labelmap`,`labeldrop`and `labelkeep` actions. The regex is anchored on both ends. To un-anchor the regex, use `.*<regex>.*`.

`<relabel_action>` determines the relabeling action to take:

- `replace`: Match `regex` against the concatenated `source_labels`. Then, set `target_label` to `replacement`, with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value. If `regex` does not match, no replacement takes place.
- `keep`: **Drop targets for which `regex` does not match the concatenated `source_labels`.**
- `drop`: **Drop targets for which `regex` matches the concatenated `source_labels`.**
- `hashmod`: Set `target_label` to the `modulus` of a hash of the concatenated `source_labels`.
- `labelmap`: Match `regex` against all label names. Then copy the values of the matching labels to label names given by `replacement` with match group references (`${1}`, `${2}`, ...) in `replacement` substituted by their value.
- `labeldrop`: Match `regex` against all label names. Any label that matches will be removed from the set of labels.
- `labelkeep`: Match `regex` against all label names. Any label that does not match will be removed from the set of labels.

Care must be taken with `labeldrop` and `labelkeep` to ensure that metrics are still uniquely labeled once the labels are removed.

### `<metric_relabel_configs>`

Metric relabeling is applied to samples as the last step before ingestion. It has the same configuration format and actions as target relabeling. Metric relabeling does not apply to automatically generated timeseries such as `up`.

One use for this is to blacklist time series that are too expensive to ingest.

### `<alert_relabel_configs>`

Alert relabeling is applied to alerts before they are sent to the Alertmanager. It has the same configuration format and actions as target relabeling. Alert relabeling is applied after external labels.

One use for this is ensuring a HA pair of Prometheus servers with different external labels send identical alerts.

### `<alertmanager_config>`

An `alertmanager_config` section specifies Alertmanager instances the Prometheus server sends alerts to. It also provides parameters to configure how to communicate with these Alertmanagers.

Alertmanagers may be statically configured via the `static_configs` parameter or dynamically discovered using one of the supported service-discovery mechanisms.

Additionally, `relabel_configs` allow selecting Alertmanagers from discovered entities and provide advanced modifications to the used API path, which is exposed through the `__alerts_path__` label.

```
# Per-target Alertmanager timeout when pushing alerts.
[ timeout: <duration> | default = 10s ]

# Prefix for the HTTP path alerts are pushed to.
[ path_prefix: <path> | default = / ]

# Configures the protocol scheme used for requests.
[ scheme: <scheme> | default = http ]

# Sets the `Authorization` header on every request with the
# configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file`.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the scrape request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# List of Azure service discovery configurations.
azure_sd_configs:
  [ - <azure_sd_config> ... ]

# List of Consul service discovery configurations.
consul_sd_configs:
  [ - <consul_sd_config> ... ]

# List of DNS service discovery configurations.
dns_sd_configs:
  [ - <dns_sd_config> ... ]

# List of EC2 service discovery configurations.
ec2_sd_configs:
  [ - <ec2_sd_config> ... ]

# List of file service discovery configurations.
file_sd_configs:
  [ - <file_sd_config> ... ]

# List of GCE service discovery configurations.
gce_sd_configs:
  [ - <gce_sd_config> ... ]

# List of Kubernetes service discovery configurations.
kubernetes_sd_configs:
  [ - <kubernetes_sd_config> ... ]

# List of Marathon service discovery configurations.
marathon_sd_configs:
  [ - <marathon_sd_config> ... ]

# List of AirBnB's Nerve service discovery configurations.
nerve_sd_configs:
  [ - <nerve_sd_config> ... ]

# List of Zookeeper Serverset service discovery configurations.
serverset_sd_configs:
  [ - <serverset_sd_config> ... ]

# List of Triton service discovery configurations.
triton_sd_configs:
  [ - <triton_sd_config> ... ]

# List of labeled statically configured Alertmanagers.
static_configs:
  [ - <static_config> ... ]

# List of Alertmanager relabel configurations.
relabel_configs:
  [ - <relabel_config> ... ]
```

### `<remote_write>`

`write_relabel_configs` is relabeling applied to samples before sending them to the remote endpoint. Write relabeling is applied after external labels. This could be used to limit which samples are sent.

There is a [small demo](https://github.com/prometheus/prometheus/blob/release-2.9/documentation/examples/remote_storage) of how to use this functionality.

```
# The URL of the endpoint to send samples to.
url: <string>

# Timeout for requests to the remote write endpoint.
[ remote_timeout: <duration> | default = 30s ]

# List of remote write relabel configurations.
write_relabel_configs:
  [ - <relabel_config> ... ]

# Sets the `Authorization` header on every remote write request with the
# configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every remote write request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file`.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every remote write request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the remote write request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# Configures the queue used to write to remote storage.
queue_config:
  # Number of samples to buffer per shard before we start dropping them.
  [ capacity: <int> | default = 10000 ]
  # Maximum number of shards, i.e. amount of concurrency.
  [ max_shards: <int> | default = 1000 ]
  # Minimum number of shards, i.e. amount of concurrency.
  [ min_shards: <int> | default = 1 ]
  # Maximum number of samples per send.
  [ max_samples_per_send: <int> | default = 100]
  # Maximum time a sample will wait in buffer.
  [ batch_send_deadline: <duration> | default = 5s ]
  # Maximum number of times to retry a batch on recoverable errors.
  [ max_retries: <int> | default = 3 ]
  # Initial retry delay. Gets doubled for every retry.
  [ min_backoff: <duration> | default = 30ms ]
  # Maximum retry delay.
  [ max_backoff: <duration> | default = 100ms ]
```

There is a list of [integrations](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage) with this feature.

### `<remote_read>`

```
# The URL of the endpoint to query from.
url: <string>

# An optional list of equality matchers which have to be
# present in a selector to query the remote read endpoint.
required_matchers:
  [ <labelname>: <labelvalue> ... ]

# Timeout for requests to the remote read endpoint.
[ remote_timeout: <duration> | default = 1m ]

# Whether reads should be made for queries for time ranges that
# the local storage should have complete data for.
[ read_recent: <boolean> | default = false ]

# Sets the `Authorization` header on every remote read request with the
# configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <string> ]
  [ password_file: <string> ]

# Sets the `Authorization` header on every remote read request with
# the configured bearer token. It is mutually exclusive with `bearer_token_file`.
[ bearer_token: <string> ]

# Sets the `Authorization` header on every remote read request with the bearer token
# read from the configured file. It is mutually exclusive with `bearer_token`.
[ bearer_token_file: /path/to/bearer/token/file ]

# Configures the remote read request's TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```

There is a list of [integrations](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage) with this feature.











# Query

最近在忙kubernetes监控的事情，需要用到prometheus的查询语句，统计相关的数据。

**官方说明**

> Prometheus provides a functional query language called **PromQL** (Prometheus Query Language) that **lets the user select and aggregate time series data in real time.** The result of an expression can either be shown as a graph, viewed as tabular data in Prometheus's expression browser, or consumed by external systems via the [HTTP API](https://prometheus.io/docs/prometheus/latest/querying/api/).



## Examples

This document is meant as a reference. For learning, it might be easier to start with a couple of [examples](https://prometheus.io/docs/prometheus/latest/querying/examples/).

## Expression language data types

In Prometheus's expression language, an expression or sub-expression can evaluate to one of four types:

- **Instant vector** - a set of time series containing a single sample for each time series, all sharing the same timestamp
- **Range vector** - a set of time series containing a range of data points over time for each time series
- **Scalar** - a simple numeric floating point value
- **String** - a simple string value; currently unused

Depending on the use-case (e.g. when graphing vs. displaying the output of an expression), **only some of these types are legal as the result from a user-specified expression**. For example, an expression that returns an instant vector is the only type that can be directly graphed.

表达语句的数据类型：

- 实例向量： 一组时序，包含各个时序的单次采样，这些时序的时间戳一样。
- 区域向量：一组时序，包含一段时间内的各个时序数据点。
- 纯量：一个浮点数值
- String：

## Literals

### String literals

**Strings may be specified as literals in single quotes, double quotes or backticks.**

PromQL follows the same [escaping rules as Go](https://golang.org/ref/spec#String_literals). In single or double quotes a backslash begins an escape sequence, which may be followed by `a`, `b`, `f`, `n`, `r`, `t`, `v` or `\`. Specific characters can be provided using octal (`\nnn`) or hexadecimal (`\xnn`, `\unnnn` and `\Unnnnnnnn`).

No escaping is processed inside backticks. Unlike Go, Prometheus does not discard newlines inside backticks.

Example:

```
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```

### Float literals

Scalar float values can be literally written as numbers of the form `[-](digits)[.(digits)]`.

```
-2.43
```

## Time series Selectors

### Instant vector selectors  （实例向量选择器）

**Instant vector selectors** allow the selection of a set of time series and a single sample value for each at a given timestamp (instant): in the simplest form, only a metric name is specified. This results in an instant vector containing elements for all time series that have this metric name.

This example selects all time series that have the `http_requests_total` metric name:

```
http_requests_total
```

**It is possible to filter these time series further by appending a set of labels to match in curly braces (`{}`).**

**可以通过{}来过滤相关的数据。**

This example selects only those time series with the `http_requests_total` metric name that also have the `job`label set to `prometheus` and their `group` label set to `canary`:

```
http_requests_total{job="prometheus",group="canary"}
```

**It is also possible to negatively match a label value, or to match label values against regular expressions. The following label matching operators exist: 可以通过一些表达式来选择相关的数据**

- `=`: Select labels that are exactly equal to the provided string.
- `!=`: Select labels that are not equal to the provided string.
- `=~`: Select labels that regex-match the provided string (or substring).
- `!~`: Select labels that do not regex-match the provided string (or substring).

For example, this selects all `http_requests_total` time series for `staging`, `testing`, and `development`environments and HTTP methods other than `GET`.

```
http_requests_total{environment=~"staging|testing|development",method!="GET"}
```

**Label matchers （标签匹配器）** that match empty label values also select all time series that do not have the specific label set at all. Regex-matches are fully anchored. It is possible to have multiple matchers for the same label name.

**Vector selectors must either specify a name or at least one label matcher that does not match the empty string.不要匹配空string** The following expression is illegal:

```
{job=~".*"} # Bad!
```

In contrast, these expressions are valid as they both have a selector that does not match empty label values.

```
{job=~".+"}              # Good!
{job=~".*",method="get"} # Good!
```

Label matchers can also be applied to metric names by matching against the internal `__name__` label. For example, the expression `http_requests_total` is equivalent to `{__name__="http_requests_total"}`. **Matchers other than `=` (`!=`, `=~`, `!~`) may also be used.** The following expression selects all metrics that have a name starting with `job:`:

```
{__name__=~"job:.*"}
```

All regular expressions in Prometheus use [RE2 syntax](https://github.com/google/re2/wiki/Syntax).

### Range Vector Selectors 区域向量选择器

Range vector literals work like instant vector literals, except that they select a range of samples back from the current instant. Syntactically, **a range duration is appended in square brackets (`[]`) at the end of a vector selector to specify how far back in time values should be fetched for each resulting range vector element.在实例向量后面加上[] 来指定过去多久的数据**

Time durations are specified as a number, followed immediately by one of the following units:

- `s` - seconds
- `m` - minutes
- `h` - hours
- `d` - days
- `w` - weeks
- `y` - years

In this example, we select all the values we have recorded within the last 5 minutes for all time series that have the metric name `http_requests_total` and a `job` label set to `prometheus`:

```
http_requests_total{job="prometheus"}[5m]
```

### Offset modifier

The `offset` modifier allows changing the time offset for individual instant and range vectors in a query.

For example, the following expression returns the value of `http_requests_total` 5 minutes in the past relative to the current query evaluation time:

```
http_requests_total offset 5m
```

**Note that the `offset` modifier always needs to follow the selector immediately, i.e. the following would be correct: 必须在selector后面**

```
sum(http_requests_total{method="GET"} offset 5m) // GOOD.
```

While the following would be *incorrect*:

```
sum(http_requests_total{method="GET"}) offset 5m // INVALID.
```

**The same works for range vectors**. **This returns the 5-minutes rate that `http_requests_total` had a week ago:**

```
rate(http_requests_total[5m] offset 1w)
```

## Subquery 子查询

Subquery **allows you to run an instant query for a given range and resolution  根据给定的range和resolution，运行一个实例查询**. The result of a subquery is a range vector.

Syntax: `<instant_query> '[' <range> ':' [<resolution>] ']' [ offset <duration> ]`

- `<resolution>` is optional. **Default is the global evaluation interval**. 默认整个所有的数据区间

## Operators 操作算子

Prometheus supports many binary and aggregation operators. These are described in detail in the [expression language operators](https://prometheus.io/docs/prometheus/latest/querying/operators/) page.

## Functions

Prometheus supports several functions to operate on data. These are described in detail in the [expression language functions](https://prometheus.io/docs/prometheus/latest/querying/functions/) page.

## Gotchas

### Staleness

When queries are run, timestamps at which to sample data are selected independently of the actual present time series data. This is mainly to support cases like aggregation (`sum`, `avg`, and so on), where multiple aggregated time series do not exactly align in time. Because of their independence, Prometheus needs to assign a value at those timestamps for each relevant time series. It does so by simply taking the newest sample before this timestamp.

If a target scrape or rule evaluation no longer returns a sample for a time series that was previously present, that time series will be marked as stale. If a target is removed, its previously returned time series will be marked as stale soon afterwards.

If a query is evaluated at a sampling timestamp after a time series is marked stale, then no value is returned for that time series. If new samples are subsequently ingested for that time series, they will be returned as normal.

If no sample is found (by default) 5 minutes before a sampling timestamp, no value is returned for that time series at this point in time. This effectively means that time series "disappear" from graphs at times where their latest collected sample is older than 5 minutes or after they are marked stale.

Staleness will not be marked for time series that have timestamps included in their scrapes. Only the 5 minute threshold will be applied in that case.

### Avoiding slow queries and overloads

If a query needs to operate on a very large amount of data, graphing it might time out or overload the server or browser. Thus, when constructing queries over unknown data, always start building the query in the tabular view of Prometheus's expression browser until the result set seems reasonable (hundreds, not thousands, of time series at most). Only when you have filtered or aggregated your data sufficiently, switch to graph mode. If the expression still takes too long to graph ad-hoc, pre-record it via a [recording rule](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/#recording-rules).

This is especially relevant for Prometheus's query language, where a bare metric name selector like `api_http_requests_total` could expand to thousands of time series with different labels. Also keep in mind that expressions which aggregate over many time series will generate load on the server even if the output is only a small number of time series. This is similar to how it would be slow to sum all values of a column in a relational database, even if the output value is only a single number.



##  Operator

## Binary operators 两元操作算子

Prometheus's query language supports basic logical and arithmetic operators. For operations between two instant vectors, the [matching behavior](https://prometheus.io/docs/prometheus/latest/querying/operators/#vector-matching) can be modified.

### Arithmetic binary operators 算数二元操作符

The following binary arithmetic operators exist in Prometheus:

- `+` (addition)
- `-` (subtraction)
- `*` (multiplication)
- `/` (division)
- `%` (modulo)
- `^` (power/exponentiation)

Binary arithmetic operators are defined between scalar/scalar, vector/scalar, and vector/vector value pairs.

**Between two scalars**, the behavior is obvious: they evaluate to another scalar that is the result of the operator applied to both scalar operands.

**Between an instant vector and a scalar**, the operator is applied to the value of every data sample in the vector. E.g. if a time series instant vector is multiplied by 2, the result is another vector in which every sample value of the original vector is multiplied by 2.

**Between two instant vectors**, a binary arithmetic operator is applied to each entry in the left-hand side vector and its [matching element](https://prometheus.io/docs/prometheus/latest/querying/operators/#vector-matching) in the right-hand vector. The result is propagated into the result vector and the metric name is dropped. Entries for which no matching entry in the right-hand vector can be found are not part of the result.

### Comparison binary operators

The following binary comparison operators exist in Prometheus:

- `==` (equal)
- `!=` (not-equal)
- `>` (greater-than)
- `<` (less-than)
- `>=` (greater-or-equal)
- `<=` (less-or-equal)

Comparison operators are defined between scalar/scalar, vector/scalar, and vector/vector value pairs. By default they filter. Their behavior can be modified by providing `bool` after the operator, which will return `0` or `1` for the value rather than filtering.

**Between two scalars**, the `bool` modifier must be provided and these operators result in another scalar that is either `0` (`false`) or `1` (`true`), depending on the comparison result.

**Between an instant vector and a scalar**, these operators are applied to the value of every data sample in the vector, and vector elements between which the comparison result is `false` get dropped from the result vector. If the `bool`modifier is provided, vector elements that would be dropped instead have the value `0` and vector elements that would be kept have the value `1`.

**Between two instant vectors**, these operators behave as a filter by default, applied to matching entries. Vector elements for which the expression is not true or which do not find a match on the other side of the expression get dropped from the result, while the others are propagated into a result vector with their original (left-hand side) metric names and label values. If the `bool` modifier is provided, vector elements that would have been dropped instead have the value `0` and vector elements that would be kept have the value `1` with the left-hand side label values.

### Logical/set binary operators

These logical/set binary operators are only defined between instant vectors:

- `and` (intersection)
- `or` (union)
- `unless` (complement)

`vector1 and vector2` results in a vector consisting of the elements of `vector1` for which there are elements in `vector2` with exactly matching label sets. Other elements are dropped. The metric name and values are carried over from the left-hand side vector.

`vector1 or vector2` results in a vector that contains all original elements (label sets + values) of `vector1` and additionally all elements of `vector2` which do not have matching label sets in `vector1`.

`vector1 unless vector2` results in a vector consisting of the elements of `vector1` for which there are no elements in `vector2` with exactly matching label sets. All matching elements in both vectors are dropped.

## Vector matching

Operations between vectors attempt to find a matching element in the right-hand side vector for each entry in the left-hand side. There are two basic types of matching behavior: One-to-one and many-to-one/one-to-many.

### One-to-one vector matches

**One-to-one** finds a unique pair of entries from each side of the operation. In the default case, that is an operation following the format `vector1 <operator> vector2`. Two entries match if they have the exact same set of labels and corresponding values. The `ignoring` keyword allows ignoring certain labels when matching, while the `on` keyword allows reducing the set of considered labels to a provided list:

```
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

Example input:

```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

Example query:

```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

This returns a result vector containing the fraction of HTTP requests with status code of 500 for each method, as measured over the last 5 minutes. Without `ignoring(code)` there would have been no match as the metrics do not share the same set of labels. The entries with methods `put` and `del` have no match and will not show up in the result:

```
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

### Many-to-one and one-to-many vector matches

**Many-to-one** and **one-to-many** matchings refer to the case where each vector element on the "one"-side can match with multiple elements on the "many"-side. This has to be explicitly requested using the `group_left` or `group_right` modifier, where left/right determines which vector has the higher cardinality.

```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

The label list provided with the group modifier contains additional labels from the "one"-side to be included in the result metrics. For `on` a label can only appear in one of the lists. Every time series of the result vector must be uniquely identifiable.

*Grouping modifiers can only be used for comparison and arithmetic. Operations as and, unless and or operations match with all possible entries in the right vector by default.*

Example query:

```yaml
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```

In this case the left vector contains more than one entry per `method` label value. Thus, we indicate this using `group_left`. The elements from the right side are now matched with multiple elements with the same `method` label on the left:

```
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

*Many-to-one and one-to-many matching are advanced use cases that should be carefully considered. Often a proper use of ignoring(<labels>) provides the desired outcome.*

## Aggregation operators 聚合算子

Prometheus supports the following built-in aggregation operators that can be used to **aggregate the elements of a single instant vector,** r**esulting in a new vector of fewer elements with aggregated values**:

- `sum` (calculate sum over dimensions)
- `min` (select minimum over dimensions)
- `max` (select maximum over dimensions)
- `avg` (calculate the average over dimensions)
- `stddev` (calculate population standard deviation over dimensions)
- `stdvar` (calculate population standard variance over dimensions)
- `count` (count number of elements in the vector)
- `count_values` (count number of elements with the same value)
- `bottomk` (smallest k elements by sample value)
- `topk` (largest k elements by sample value)
- `quantile` (calculate φ-quantile (0 ≤ φ ≤ 1) over dimensions)

These operators can either be used to aggregate over **all** label dimensions or preserve distinct dimensions by including a `without` or `by` clause.

```yaml
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

`parameter` is only required for `count_values`, `quantile`, `topk` and `bottomk`. `without` **removes the listed labels from the result vector**, while all other labels are preserved the output. `by` does the opposite and drops labels that are not listed in the `by` clause, even if their label values are identical between all elements of the vector.

`count_values` outputs one time series per unique sample value. Each series has an additional label. The name of that label is given by the aggregation parameter, and the label value is the unique sample value. The value of each time series is the number of times that sample value was present.

`topk` and `bottomk` are different from other aggregators in that a subset of the input samples, including the original labels, are returned in the result vector. `by` and `without` are only used to bucket the input vector.

Example:

If the metric `http_requests_total` had time series that fan out by `application`, `instance`, and `group` labels, we could calculate the total number of seen HTTP requests per application and group over all instances via:

```yaml
sum(http_requests_total) without (instance)
```

Which is equivalent to:

```yaml
 sum(http_requests_total) by (application, group)
```

If we are just interested in the total of HTTP requests we have seen in **all** applications, we could simply write:

```
sum(http_requests_total)
```

To count the number of binaries running each build version we could write:

```
count_values("version", build_version)
```

To get the 5 largest HTTP requests counts across all instances we could write:

```
topk(5, http_requests_total)
```

## Binary operator precedence

The following list shows the precedence of binary operators in Prometheus, from highest to lowest.

1. `^`
2. `*`, `/`, `%`
3. `+`, `-`
4. `==`, `!=`, `<=`, `<`, `>=`, `>`
5. `and`, `unless`
6. `or`

Operators on the same precedence level are left-associative. For example, `2 * 3 % 2` is equivalent to `(2 * 3) % 2`. However `^` is right associative, so `2 ^ 3 ^ 2` is equivalent to `2 ^ (3 ^ 2)`.





# FUNCTIONS



Some functions have default arguments, e.g. `year(v=vector(time()) instant-vector)`. This means that there is one argument `v` which is an instant vector, which if not provided it will default to the value of the expression `vector(time())`.

## `abs()`

`abs(v instant-vector)` returns the input vector with all sample values converted to their absolute value.

## `absent()`

`absent(v instant-vector)` returns an empty vector if the vector passed to it has any elements and a 1-element vector with the value 1 if the vector passed to it has no elements.

This is useful for alerting on when no time series exist for a given metric name and label combination.

```
absent(nonexistent{job="myjob"})
# => {job="myjob"}

absent(nonexistent{job="myjob",instance=~".*"})
# => {job="myjob"}

absent(sum(nonexistent{job="myjob"}))
# => {}
```

In the second example, `absent()` tries to be smart about deriving labels of the 1-element output vector from the input vector.

## `ceil()`

`ceil(v instant-vector)` rounds the sample values of all elements in `v` up to the nearest integer.

## `changes()`

For each input time series, `changes(v range-vector)` returns the number of times its value has changed within the provided time range as an instant vector.

## `clamp_max()`

`clamp_max(v instant-vector, max scalar)` clamps the sample values of all elements in `v` to have an upper limit of `max`.

## `clamp_min()`

`clamp_min(v instant-vector, min scalar)` clamps the sample values of all elements in `v` to have a lower limit of `min`.

## `day_of_month()`

`day_of_month(v=vector(time()) instant-vector)` returns the day of the month for each of the given times in UTC. Returned values are from 1 to 31.

## `day_of_week()`

`day_of_week(v=vector(time()) instant-vector)` returns the day of the week for each of the given times in UTC. Returned values are from 0 to 6, where 0 means Sunday etc.

## `days_in_month()`

`days_in_month(v=vector(time()) instant-vector)` returns number of days in the month for each of the given times in UTC. Returned values are from 28 to 31.

## `delta()`

`delta(v range-vector)` calculates the difference between the first and last value of each time series element in a range vector `v`, returning an instant vector with the given deltas and equivalent labels. The delta is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if the sample values are all integers.

The following example expression returns the difference in CPU temperature between now and 2 hours ago:

```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

`delta` should only be used with gauges.

## `deriv()`

`deriv(v range-vector)` calculates the per-second derivative of the time series in a range vector `v`, using [simple linear regression](https://en.wikipedia.org/wiki/Simple_linear_regression).

`deriv` should only be used with gauges.

## `exp()`

`exp(v instant-vector)` calculates the exponential function for all elements in `v`. Special cases are:

- `Exp(+Inf) = +Inf`
- `Exp(NaN) = NaN`

## `floor()`

`floor(v instant-vector)` rounds the sample values of all elements in `v` down to the nearest integer.

## `histogram_quantile()`

`histogram_quantile(φ float, b instant-vector)` calculates the φ-quantile (0 ≤ φ ≤ 1) from the buckets `b` of a[histogram](https://prometheus.io/docs/concepts/metric_types/#histogram). (See [histograms and summaries](https://prometheus.io/docs/practices/histograms) for a detailed explanation of φ-quantiles and the usage of the histogram metric type in general.) The samples in `b` are the counts of observations in each bucket. Each sample must have a label `le` where the label value denotes the inclusive upper bound of the bucket. (Samples without such a label are silently ignored.) The [histogram metric type](https://prometheus.io/docs/concepts/metric_types/#histogram) automatically provides time series with the `_bucket` suffix and the appropriate labels.

Use the `rate()` function to specify the time window for the quantile calculation.

Example: A histogram metric is called `http_request_duration_seconds`. To calculate the 90th percentile of request durations over the last 10m, use the following expression:

```
histogram_quantile(0.9, rate(http_request_duration_seconds_bucket[10m]))
```

The quantile is calculated for each label combination in `http_request_duration_seconds`. To aggregate, use the `sum()` aggregator around the `rate()` function. Since the `le` label is required by `histogram_quantile()`, it has to be included in the `by` clause. The following expression aggregates the 90th percentile by `job`:

```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (job, le))
```

To aggregate everything, specify only the `le` label:

```
histogram_quantile(0.9, sum(rate(http_request_duration_seconds_bucket[10m])) by (le))
```

The `histogram_quantile()` function interpolates quantile values by assuming a linear distribution within a bucket. The highest bucket must have an upper bound of `+Inf`. (Otherwise, `NaN` is returned.) If a quantile is located in the highest bucket, the upper bound of the second highest bucket is returned. A lower limit of the lowest bucket is assumed to be 0 if the upper bound of that bucket is greater than 0. In that case, the usual linear interpolation is applied within that bucket. Otherwise, the upper bound of the lowest bucket is returned for quantiles located in the lowest bucket.

If `b` contains fewer than two buckets, `NaN` is returned. For φ < 0, `-Inf` is returned. For φ > 1, `+Inf` is returned.

## `holt_winters()`

`holt_winters(v range-vector, sf scalar, tf scalar)` produces a smoothed value for time series based on the range in `v`. The lower the smoothing factor `sf`, the more importance is given to old data. The higher the trend factor `tf`, the more trends in the data is considered. Both `sf` and `tf` must be between 0 and 1.

`holt_winters` should only be used with gauges.

## `hour()`

`hour(v=vector(time()) instant-vector)` returns the hour of the day for each of the given times in UTC. Returned values are from 0 to 23.

## `idelta()`

```
idelta(v range-vector)
```

`idelta(v range-vector)` calculates the difference between the last two samples in the range vector `v`, returning an instant vector with the given deltas and equivalent labels.

`idelta` should only be used with gauges.

## `increase()`

`increase(v range-vector)` calculates the increase in the time series in the range vector. Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for. The increase is extrapolated to cover the full time range as specified in the range vector selector, so that it is possible to get a non-integer result even if a counter increases only by integer increments.

The following example expression returns the number of HTTP requests as measured over the last 5 minutes, per time series in the range vector:

```
increase(http_requests_total{job="api-server"}[5m])
```

`increase` should only be used with counters. It is syntactic sugar for `rate(v)` multiplied by the number of seconds under the specified time range window, and should be used primarily for human readability. Use `rate` in recording rules so that increases are tracked consistently on a per-second basis.

## `irate()`

`irate(v range-vector)` calculates the per-second instant rate of increase of the time series in the range vector. This is based on the last two data points. Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for.

The following example expression returns the per-second rate of HTTP requests looking up to 5 minutes back for the two most recent data points, per time series in the range vector:

```
irate(http_requests_total{job="api-server"}[5m])
```

`irate` should only be used when graphing volatile, fast-moving counters. Use `rate` for alerts and slow-moving counters, as brief changes in the rate can reset the `FOR` clause and graphs consisting entirely of rare spikes are hard to read.

Note that when combining `irate()` with an [aggregation operator](https://prometheus.io/docs/prometheus/latest/querying/operators/#aggregation-operators) (e.g. `sum()`) or a function aggregating over time (any function ending in `_over_time`), always take a `irate()` first, then aggregate. Otherwise `irate()` cannot detect counter resets when your target restarts.

## `label_join()`

For each timeseries in `v`, `label_join(v instant-vector, dst_label string, separator string, src_label_1 string, src_label_2 string, ...)` joins all the values of all the `src_labels` using `separator`and returns the timeseries with the label `dst_label` containing the joined value. There can be any number of `src_labels` in this function.

This example will return a vector with each time series having a `foo` label with the value `a,b,c` added to it:

```
label_join(up{job="api-server",src1="a",src2="b",src3="c"}, "foo", ",", "src1", "src2", "src3")
```

## `label_replace()`

For each timeseries in `v`, `label_replace(v instant-vector, dst_label string, replacement string, src_label string, regex string)` matches the regular expression `regex` against the label `src_label`. If it matches, then the timeseries is returned with the label `dst_label` replaced by the expansion of `replacement`. `$1` is replaced with the first matching subgroup, `$2` with the second etc. If the regular expression doesn't match then the timeseries is returned unchanged.

This example will return a vector with each time series having a `foo` label with the value `a` added to it:

```
label_replace(up{job="api-server",service="a:c"}, "foo", "$1", "service", "(.*):.*")
```

## `ln()`

`ln(v instant-vector)` calculates the natural logarithm for all elements in `v`. Special cases are:

- `ln(+Inf) = +Inf`
- `ln(0) = -Inf`
- `ln(x < 0) = NaN`
- `ln(NaN) = NaN`

## `log2()`

`log2(v instant-vector)` calculates the binary logarithm for all elements in `v`. The special cases are equivalent to those in `ln`.

## `log10()`

`log10(v instant-vector)` calculates the decimal logarithm for all elements in `v`. The special cases are equivalent to those in `ln`.

## `minute()`

`minute(v=vector(time()) instant-vector)` returns the minute of the hour for each of the given times in UTC. Returned values are from 0 to 59.

## `month()`

`month(v=vector(time()) instant-vector)` returns the month of the year for each of the given times in UTC. Returned values are from 1 to 12, where 1 means January etc.

## `predict_linear()`

`predict_linear(v range-vector, t scalar)` predicts the value of time series `t` seconds from now, based on the range vector `v`, using [simple linear regression](https://en.wikipedia.org/wiki/Simple_linear_regression).

`predict_linear` should only be used with gauges.

## `rate()`

`rate(v range-vector)` **calculates the per-second average rate of increase of the time series in the range vector.** Breaks in monotonicity (such as counter resets due to target restarts) are automatically adjusted for. Also, the calculation extrapolates to the ends of the time range, allowing for missed scrapes or imperfect alignment of scrape cycles with the range's time period.

The following example expressio**n returns the per-second rate of HTTP requests as measured over the last 5 minutes**, per time series in the range vector:

```
rate(http_requests_total{job="api-server"}[5m])
```

`rate` should only be used with counters. It is best suited for alerting, and for graphing of slow-moving counters.

Note that when combining `rate()` with an aggregation operator (e.g. `sum()`) or a function aggregating over time (any function ending in `_over_time`), always take a `rate()` first, then aggregate. Otherwise `rate()` cannot detect counter resets when your target restarts.

## `resets()`

For each input time series, `resets(v range-vector)` returns the number of counter resets within the provided time range as an instant vector. Any decrease in the value between two consecutive samples is interpreted as a counter reset.

`resets` should only be used with counters.

## `round()`

`round(v instant-vector, to_nearest=1 scalar)` rounds the sample values of all elements in `v` to the nearest integer. Ties are resolved by rounding up. The optional `to_nearest` argument allows specifying the nearest multiple to which the sample values should be rounded. This multiple may also be a fraction.

## `scalar()`

Given a single-element input vector, `scalar(v instant-vector)` returns the sample value of that single element as a scalar. If the input vector does not have exactly one element, `scalar` will return `NaN`.

## `sort()`

`sort(v instant-vector)` returns vector elements sorted by their sample values, in ascending order.

## `sort_desc()`

Same as `sort`, but sorts in descending order.

## `sqrt()`

`sqrt(v instant-vector)` calculates the square root of all elements in `v`.

## `time()`

`time()` returns the number of seconds since January 1, 1970 UTC. Note that this does not actually return the current time, but the time at which the expression is to be evaluated.

## `timestamp()`

`timestamp(v instant-vector)` returns the timestamp of each of the samples of the given vector as the number of seconds since January 1, 1970 UTC.

*This function was added in Prometheus 2.0*

## `vector()`

`vector(s scalar)` returns the scalar `s` as a vector with no labels.

## `year()`

`year(v=vector(time()) instant-vector)` returns the year for each of the given times in UTC.

## `<aggregation>_over_time()`

The following functions allow aggregating each series of a given range vector over time and return an instant vector with per-series aggregation results:

- `avg_over_time(range-vector)`: the average value of all points in the specified interval.
- `min_over_time(range-vector)`: the minimum value of all points in the specified interval.
- `max_over_time(range-vector)`: the maximum value of all points in the specified interval.
- `sum_over_time(range-vector)`: the sum of all values in the specified interval.
- `count_over_time(range-vector)`: the count of all values in the specified interval.
- `quantile_over_time(scalar, range-vector)`: the φ-quantile (0 ≤ φ ≤ 1) of the values in the specified interval.
- `stddev_over_time(range-vector)`: the population standard deviation of the values in the specified interval.
- `stdvar_over_time(range-vector)`: the population standard variance of the values in the specified interval.

Note that all values in the specified interval have the same weight in the aggregation even if the values are not equally spaced throughout the interval.





# 最佳实践

## METRIC AND LABEL NAMING

The metric and label conventions presented in this document are not required for using Prometheus, but can serve as both a style-guide and a collection of best practices. Individual organizations may want to approach some of these practices, e.g. naming conventions, differently.

## Metric names

A metric name...

- ...must comply with the [data model](https://prometheus.io/docs/concepts/data_model/#metric-names-and-labels) for valid characters.

- ...should have a (single-word) application prefix relevant to the domain the metric belongs to. The prefix is sometimes referred to as `namespace`  by client libraries. For metrics specific to an application, the prefix is usually the application name itself. Sometimes, however, metrics are more generic, like standardized metrics exported by client libraries. Examples:

  - `**prometheus**_notifications_total` (specific to the Prometheus server)
  - `**process**_cpu_seconds_total` (exported by many client libraries)
  - `**http**_request_duration_seconds` (for all HTTP requests)

- ...must have a single unit (i.e. do not mix seconds with milliseconds, or seconds with bytes).

- ...should use base units (e.g. seconds, bytes, meters - not milliseconds, megabytes, kilometers). See below for a list of base units.

- ...should have a suffix describing the unit, in plural form. Note that an accumulating count has

   

  ```
  total
  ```

   

  as a suffix, in addition to the unit if applicable.

  - `http_request_duration_**seconds**`
  - `node_memory_usage_**bytes**`
  - `http_requests_**total**` (for a unit-less accumulating count)
  - `process_cpu_**seconds_total**` (for an accumulating count with unit)

- ...should represent the same logical thing-being-measured across all label dimensions.

  - request duration
  - bytes of data transfer
  - instantaneous resource usage as a percentage

As a rule of thumb, either the `sum()` or the `avg()` over all dimensions of a given metric should be meaningful (though not necessarily useful). If it is not meaningful, split the data up into multiple metrics. For example, having the capacity of various queues in one metric is good, while mixing the capacity of a queue with the current number of elements in the queue is not.

## Labels

Use labels to differentiate the characteristics of the thing that is being measured:

- `api_http_requests_total` - differentiate request types: `type="create|update|delete"`
- `api_request_duration_seconds` - differentiate request stages: `stage="extract|transform|load"`

Do not put the label names in the metric name, as this introduces redundancy and will cause confusion if the respective labels are aggregated away.

**CAUTION:** Remember that every unique combination of key-value label pairs represents a new time series, which can dramatically increase the amount of data stored. Do not use labels to store dimensions with high cardinality (many different label values), such as user IDs, email addresses, or other unbounded sets of values.

## Base units

Prometheus does not have any units hard coded. For better compatibility, base units should be used. The following lists some metrics families with their base unit. The list is not exhaustive.

| Family           | Base unit | Remark                                                       |
| :--------------- | :-------- | :----------------------------------------------------------- |
| Time             | seconds   |                                                              |
| Temperature      | celsius   | Celsius is the most common one encountered in practice       |
| Length           | meters    |                                                              |
| Bytes            | bytes     |                                                              |
| Bits             | bytes     |                                                              |
| Percent          | ratio(*)  | Values are 0-1.  *) Usually 'ratio' is not used as suffix, but rather A_per_B. Exceptions are e.g. disk_usage_ratio |
| Voltage          | volts     |                                                              |
| Electric current | amperes   |                                                              |
| Energy           | joules    |                                                              |
| Weight           | grams     |                                                              |



# Alert Manager

不用这么麻烦，可以使用garafana的alert机制，可以直接通知到丁丁。

:dragon_face: 啪啪打脸了，garafna不支持template value的alert报警查询。

那么，我们继续搞 alert manager



[Alertmanager](https://github.com/prometheus/alertmanager) is configured via command-line flags and a configuration file. While the command-line flags configure immutable system parameters, the configuration file defines inhibition rules, notification routing and notification receivers.

The [visual editor](https://prometheus.io/webtools/alerting/routing-tree-editor) can assist in building routing trees.

To view all available command-line flags, run `alertmanager -h`.

Alertmanager can reload its configuration at runtime. If the new configuration is not well-formed, the changes will not be applied and an error is logged. A configuration reload is triggered by sending a `SIGHUP` to the process or sending a HTTP POST request to the `/-/reload` endpoint.

## Configuration file

To specify which configuration file to load, use the `--config.file`flag.

```yaml
./alertmanager --config.file=simple.yml
```

The file is written in the [YAML format](http://en.wikipedia.org/wiki/YAML), defined by the scheme described below. Brackets indicate that a parameter is optional. For non-list parameters the value is set to the specified default.

Generic placeholders are defined as follows:

- `<duration>`: a duration matching the regular expression `[0-9]+(ms|[smhdwy])`
- `<labelname>`: a string matching the regular expression `[a-zA-Z_][a-zA-Z0-9_]*`
- `<labelvalue>`: a string of unicode characters
- `<filepath>`: a valid path in the current working directory
- `<boolean>`: a boolean that can take the values `true` or `false`
- `<string>`: a regular string
- `<secret>`: a regular string that is a secret, such as a password
- `<tmpl_string>`: a string which is template-expanded before usage
- `<tmpl_secret>`: a string which is template-expanded before usage that is a secret

The other placeholders are specified separately.

A valid example file can be found [here](https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml).

The global configuration specifies parameters that are valid in all other configuration contexts. They also serve as defaults for other configuration sections.

```yaml
global:
  # ResolveTimeout is the time after which an alert is declared resolved
  # if it has not been updated.
  [ resolve_timeout: <duration> | default = 5m ]

  # The default SMTP From header field.
  [ smtp_from: <tmpl_string> ]
  # The default SMTP smarthost used for sending emails, including port number.
  # Port number usually is 25, or 587 for SMTP over TLS (sometimes referred to as STARTTLS).
  # Example: smtp.example.org:587
  [ smtp_smarthost: <string> ]
  # The default hostname to identify to the SMTP server.
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  # SMTP Auth using LOGIN and PLAIN.
  [ smtp_auth_password: <secret> ]
  # SMTP Auth using PLAIN.
  [ smtp_auth_identity: <string> ]
  # SMTP Auth using CRAM-MD5. 
  [ smtp_auth_secret: <secret> ]
  # The default SMTP TLS requirement.
  [ smtp_require_tls: <bool> | default = true ]

  # The API URL to use for Slack notifications.
  [ slack_api_url: <secret> ]
  [ victorops_api_key: <secret> ]
  [ victorops_api_url: <string> | default = "https://alert.victorops.com/integrations/generic/20131114/alert/" ]
  [ pagerduty_url: <string> | default = "https://events.pagerduty.com/v2/enqueue" ]
  [ opsgenie_api_key: <secret> ]
  [ opsgenie_api_url: <string> | default = "https://api.opsgenie.com/" ]
  [ hipchat_api_url: <string> | default = "https://api.hipchat.com/" ]
  [ hipchat_auth_token: <secret> ]
  [ wechat_api_url: <string> | default = "https://qyapi.weixin.qq.com/cgi-bin/" ]
  [ wechat_api_secret: <secret> ]
  [ wechat_api_corp_id: <string> ]

  # The default HTTP client configuration
  [ http_config: <http_config> ]

# Files from which custom notification template definitions are read.
# The last component may use a wildcard matcher, e.g. 'templates/*.tmpl'.
# 自定义通知模版
templates:
  [ - <filepath> ... ]

# The root node of the routing tree.
# 路由树的根节点
route: <route>

# A list of notification receivers.
# 消息通知的接受者
receivers:
  - <receiver> ...

# A list of inhibition rules.
# 抑制规则列表
inhibit_rules:
  [ - <inhibit_rule> ... ]
```

## 路由配置 `<route>`

A **route block** defines **a node in a routing tree** and its children. Its optional configuration parameters are inherited from its parent node if not set.

**route 块定义了路由树的一个节点以及它的子节点**。

Every alert **enters the routing tree at the configured top-level route,** which must match all alerts (i.e. not have any configured matchers). **It then traverses the child nodes**. If `continue` is set to false, it **stops after the first matching child.** If `continue` is true on a matching node, **the alert will continue matching against subsequent siblings**. If an alert does not match any children of a node (no matching child nodes, or none exist), the alert is handled based on the configuration parameters of the current node.

每个Alert进入到路由树的根节点。

```yaml
[ receiver: <string> ]
# The labels by which incoming alerts are grouped together. For example,
# multiple alerts coming in for cluster=A and alertname=LatencyHigh would
# be batched into a single group.
#
# To aggregate by all possible labels use the special value '...' as the sole label name, for example:
# group_by: ['...'] 
# This effectively disables aggregation entirely, passing through all 
# alerts as-is. This is unlikely to be what you want, unless you have 
# a very low alert volume or your upstream notification system performs 
# its own grouping.
[ group_by: '[' <labelname>, ... ']' ]

# Whether an alert should continue matching subsequent sibling nodes.
[ continue: <boolean> | default = false ]

# A set of equality matchers an alert has to fulfill to match the node.
match:
  [ <labelname>: <labelvalue>, ... ]

# A set of regex-matchers an alert has to fulfill to match the node.
match_re:
  [ <labelname>: <regex>, ... ]

# How long to initially wait to send a notification for a group
# of alerts. Allows to wait for an inhibiting alert to arrive or collect
# more initial alerts for the same group. (Usually ~0s to few minutes.)
[ group_wait: <duration> | default = 30s ]

# How long to wait before sending a notification about new alerts that
# are added to a group of alerts for which an initial notification has
# already been sent. (Usually ~5m or more.)
[ group_interval: <duration> | default = 5m ]

# How long to wait before sending a notification again if it has already
# been sent successfully for an alert. (Usually ~3h or more).
[ repeat_interval: <duration> | default = 4h ]

# Zero or more child routes.
routes:
  [ - <route> ... ]
```

### Example

```yaml
# The root route with all parameters, which are inherited by the child
# routes if they are not overwritten.
route:
  receiver: 'default-receiver'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: [cluster, alertname]
  # All alerts that do not match the following child routes
  # will remain at the root node and be dispatched to 'default-receiver'.
  # 如果所有的子路由都没有匹配到，将保留在root节点，并发送到默认的receiver。
  routes:
  # All alerts with service=mysql or service=cassandra
  # are dispatched to the database pager.
  # 所有匹配到  service=mysql的标签，将发送到 database-pager
  - receiver: 'database-pager'
    group_wait: 10s
    match_re:
      service: mysql|cassandra
  # All alerts with the team=frontend label match this sub-route.
  # They are grouped by product and environment rather than cluster
  # and alertname.
  - receiver: 'frontend-pager'
    group_by: [product, environment]
    match:
      team: frontend
```

## 约束规则 `<inhibit_rule>`

An inhibition rule mutes an alert (target) matching a set of matchers when an alert (source) exists that matches another set of matchers. Both target and source alerts must have the same label values for the label names in the `equal` list.

Semantically, **a missing label** and **a label with an empty value** are the same thing. Therefore, if all the label names listed in `equal` are missing from both the source and target alerts, the inhibition rule will apply.

To prevent an alert from inhibiting itself, an alert that matches *both* the target and the source side of a rule cannot be inhibited by alerts for which the same is true (including itself). However, we recommend to choose target and source matchers in a way that alerts never match both sides. It is much easier to reason about and does not trigger this special case.

```yaml
# Matchers that have to be fulfilled in the alerts to be muted.
target_match:
  [ <labelname>: <labelvalue>, ... ]
target_match_re:
  [ <labelname>: <regex>, ... ]

# Matchers for which one or more alerts have to exist for the
# inhibition to take effect.
source_match:
  [ <labelname>: <labelvalue>, ... ]
source_match_re:
  [ <labelname>: <regex>, ... ]

# Labels that must have an equal value in the source and target
# alert for the inhibition to take effect.
[ equal: '[' <labelname>, ... ']' ]
```

## `<http_config>`

A `http_config` allows configuring the HTTP client that the receiver uses to communicate with HTTP-based API services.

```yaml
# Note that `basic_auth`, `bearer_token` and `bearer_token_file` options are
# mutually exclusive.

# Sets the `Authorization` header with the configured username and password.
# password and password_file are mutually exclusive.
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

# Sets the `Authorization` header with the configured bearer token.
[ bearer_token: <secret> ]

# Sets the `Authorization` header with the bearer token read from the configured file.
[ bearer_token_file: <filepath> ]

# Configures the TLS settings.
tls_config:
  [ <tls_config> ]

# Optional proxy URL.
[ proxy_url: <string> ]
```

## `<tls_config>`

A `tls_config` allows configuring TLS connections.

```yaml
# CA certificate to validate the server certificate with.
[ ca_file: <filepath> ]

# Certificate and key files for client cert authentication to the server.
[ cert_file: <filepath> ]
[ key_file: <filepath> ]

# ServerName extension to indicate the name of the server.
# http://tools.ietf.org/html/rfc4366#section-3.1
[ server_name: <string> ]

# Disable validation of the server certificate.
[ insecure_skip_verify: <boolean> | default = false]
```

## `<receiver>`

Receiver is a named configuration of one or more notification integrations.

**We're not actively adding new receivers, we recommend implementing custom notification integrations via the webhook receiver.**

```yaml
# The unique name of the receiver.
name: <string>

# Configurations for several notification integrations.
email_configs:
  [ - <email_config>, ... ]
hipchat_configs:
  [ - <hipchat_config>, ... ]
pagerduty_configs:
  [ - <pagerduty_config>, ... ]
pushover_configs:
  [ - <pushover_config>, ... ]
slack_configs:
  [ - <slack_config>, ... ]
opsgenie_configs:
  [ - <opsgenie_config>, ... ]
webhook_configs:
  [ - <webhook_config>, ... ]
victorops_configs:
  [ - <victorops_config>, ... ]
wechat_configs:
  [ - <wechat_config>, ... ]
```

## `<email_config>`

```yaml
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = false ]

# The email address to send notifications to.
to: <tmpl_string>

# The sender address.
[ from: <tmpl_string> | default = global.smtp_from ]

# The SMTP host through which emails are sent.
[ smarthost: <string> | default = global.smtp_smarthost ]

# The hostname to identify to the SMTP server.
[ hello: <string> | default = global.smtp_hello ]

# SMTP authentication information.
[ auth_username: <string> | default = global.smtp_auth_username ]
[ auth_password: <secret> | default = global.smtp_auth_password ]
[ auth_secret: <secret> | default = global.smtp_auth_secret ]
[ auth_identity: <string> | default = global.smtp_auth_identity ]

# The SMTP TLS requirement.
[ require_tls: <bool> | default = global.smtp_require_tls ]

# TLS configuration.
tls_config:
  [ <tls_config> ]

# The HTML body of the email notification.
[ html: <tmpl_string> | default = '{{ template "email.default.html" . }}' ]
# The text body of the email notification.
[ text: <tmpl_string> ]

# Further headers email header key/value pairs. Overrides any headers
# previously set by the notification implementation.
[ headers: { <string>: <tmpl_string>, ... } ]
```

## `<hipchat_config>`

HipChat notifications use a [Build Your Own](https://confluence.atlassian.com/hc/integrations-with-hipchat-server-683508267.html) integration.

```yaml
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = false ]

# The HipChat Room ID.
room_id: <tmpl_string>
# The auth token.
[ auth_token: <secret> | default = global.hipchat_auth_token ]
# The URL to send API requests to.
[ api_url: <string> | default = global.hipchat_api_url ]

# See https://www.hipchat.com/docs/apiv2/method/send_room_notification
# A label to be shown in addition to the sender's name.
[ from:  <tmpl_string> | default = '{{ template "hipchat.default.from" . }}' ]
# The message body.
[ message:  <tmpl_string> | default = '{{ template "hipchat.default.message" . }}' ]
# Whether this message should trigger a user notification.
[ notify:  <boolean> | default = false ]
# Determines how the message is treated by the alertmanager and rendered inside HipChat. Valid values are 'text' and 'html'.
[ message_format:  <string> | default = 'text' ]
# Background color for message.
[ color:  <tmpl_string> | default = '{{ if eq .Status "firing" }}red{{ else }}green{{ end }}' ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

## `<pagerduty_config>`

PagerDuty notifications are sent via the [PagerDuty API](https://developer.pagerduty.com/documentation/integration/events). PagerDuty provides documentation on how to integrate [here](https://www.pagerduty.com/docs/guides/prometheus-integration-guide/).

```yaml
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

# The following two options are mutually exclusive.
# The PagerDuty integration key (when using PagerDuty integration type `Events API v2`).
routing_key: <tmpl_secret>
# The PagerDuty integration key (when using PagerDuty integration type `Prometheus`).
service_key: <tmpl_secret>

# The URL to send API requests to
[ url: <string> | default = global.pagerduty_url ]

# The client identification of the Alertmanager.
[ client:  <tmpl_string> | default = '{{ template "pagerduty.default.client" . }}' ]
# A backlink to the sender of the notification.
[ client_url:  <tmpl_string> | default = '{{ template "pagerduty.default.clientURL" . }}' ]

# A description of the incident.
[ description: <tmpl_string> | default = '{{ template "pagerduty.default.description" .}}' ]

# Severity of the incident.
[ severity: <tmpl_string> | default = 'error' ]

# A set of arbitrary key/value pairs that provide further detail
# about the incident.
[ details: { <string>: <tmpl_string>, ... } | default = {
  firing:       '{{ template "pagerduty.default.instances" .Alerts.Firing }}'
  resolved:     '{{ template "pagerduty.default.instances" .Alerts.Resolved }}'
  num_firing:   '{{ .Alerts.Firing | len }}'
  num_resolved: '{{ .Alerts.Resolved | len }}'
} ]

# Images to attach to the incident.
images:
  [ <image_config> ... ]

# Links to attach to the incident.
links:
  [ <link_config> ... ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

### `<image_config>`

The fields are documented in the [PagerDuty API documentation](https://v2.developer.pagerduty.com/v2/docs/send-an-event-events-api-v2#section-the-images-property).

```yaml
source: <tmpl_string>
alt: <tmpl_string>
text: <tmpl_string>
```

### `<link_config>`

The fields are documented in the [PagerDuty API documentation](https://v2.developer.pagerduty.com/v2/docs/send-an-event-events-api-v2#section-the-links-property).

```
href: <tmpl_string>
text: <tmpl_string>
```

## `<pushover_config>`

Pushover notifications are sent via the [Pushover API](https://pushover.net/api).

```
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

# The recipient user’s user key.
user_key: <secret>

# Your registered application’s API token, see https://pushover.net/apps
token: <secret>

# Notification title.
[ title: <tmpl_string> | default = '{{ template "pushover.default.title" . }}' ]

# Notification message.
[ message: <tmpl_string> | default = '{{ template "pushover.default.message" . }}' ]

# A supplementary URL shown alongside the message.
[ url: <tmpl_string> | default = '{{ template "pushover.default.url" . }}' ]

# Priority, see https://pushover.net/api#priority
[ priority: <tmpl_string> | default = '{{ if eq .Status "firing" }}2{{ else }}0{{ end }}' ]

# How often the Pushover servers will send the same notification to the user.
# Must be at least 30 seconds.
[ retry: <duration> | default = 1m ]

# How long your notification will continue to be retried for, unless the user
# acknowledges the notification.
[ expire: <duration> | default = 1h ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

## `<slack_config>`

Slack notifications are sent via [Slack webhooks](https://api.slack.com/incoming-webhooks). The notification contains an [attachment](https://api.slack.com/docs/message-attachments).

```
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = false ]

# The Slack webhook URL.
[ api_url: <secret> | default = global.slack_api_url ]

# The channel or user to send notifications to.
channel: <tmpl_string>

# API request data as defined by the Slack webhook API.
[ icon_emoji: <tmpl_string> ]
[ icon_url: <tmpl_string> ]
[ link_names: <boolean> | default = false ]
[ username: <tmpl_string> | default = '{{ template "slack.default.username" . }}' ]
# The following parameters define the attachment.
actions:
  [ <action_config> ... ]
[ callback_id: <tmpl_string> | default = '{{ template "slack.default.callbackid" . }}' ]
[ color: <tmpl_string> | default = '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}' ]
[ fallback: <tmpl_string> | default = '{{ template "slack.default.fallback" . }}' ]
fields:
  [ <field_config> ... ]
[ footer: <tmpl_string> | default = '{{ template "slack.default.footer" . }}' ]
[ pretext: <tmpl_string> | default = '{{ template "slack.default.pretext" . }}' ]
[ short_fields: <boolean> | default = false ]
[ text: <tmpl_string> | default = '{{ template "slack.default.text" . }}' ]
[ title: <tmpl_string> | default = '{{ template "slack.default.title" . }}' ]
[ title_link: <tmpl_string> | default = '{{ template "slack.default.titlelink" . }}' ]
[ image_url: <tmpl_string> ]
[ thumb_url: <tmpl_string> ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

### `<action_config>`

The fields are documented in the [Slack API documentation](https://api.slack.com/docs/message-attachments#action_fields).

```
type: <tmpl_string>
text: <tmpl_string>
url: <tmpl_string>
[ style: <tmpl_string> [ default = '' ]
```

### `<field_config>`

The fields are documented in the [Slack API documentation](https://api.slack.com/docs/message-attachments#fields).

```
title: <tmpl_string>
value: <tmpl_string>
[ short: <boolean> | default = slack_config.short_fields ]
```

## `<opsgenie_config>`

OpsGenie notifications are sent via the [OpsGenie API](https://docs.opsgenie.com/docs/alert-api).

```
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

# The API key to use when talking to the OpsGenie API.
[ api_key: <secret> | default = global.opsgenie_api_key ]

# The host to send OpsGenie API requests to.
[ api_url: <string> | default = global.opsgenie_api_url ]

# Alert text limited to 130 characters.
[ message: <tmpl_string> ]

# A description of the incident.
[ description: <tmpl_string> | default = '{{ template "opsgenie.default.description" . }}' ]

# A backlink to the sender of the notification.
[ source: <tmpl_string> | default = '{{ template "opsgenie.default.source" . }}' ]

# A set of arbitrary key/value pairs that provide further detail
# about the incident.
[ details: { <string>: <tmpl_string>, ... } ]

# Comma separated list of team responsible for notifications.
[ teams: <tmpl_string> ]

# Comma separated list of tags attached to the notifications.
[ tags: <tmpl_string> ]

# Additional alert note.
[ note: <tmpl_string> ]

# Priority level of alert. Possible values are P1, P2, P3, P4, and P5.
[ priority: <tmpl_string> ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

## `<victorops_config>`

VictorOps notifications are sent out via the [VictorOps API](https://help.victorops.com/knowledge-base/victorops-restendpoint-integration/)

```
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

# The API key to use when talking to the VictorOps API.
[ api_key: <secret> | default = global.victorops_api_key ]

# The VictorOps API URL.
[ api_url: <string> | default = global.victorops_api_url ]

# A key used to map the alert to a team.
routing_key: <tmpl_string>

# Describes the behavior of the alert (CRITICAL, WARNING, INFO).
[ message_type: <tmpl_string> | default = 'CRITICAL' ]

# Contains summary of the alerted problem.
[ entity_display_name: <tmpl_string> | default = '{{ template "victorops.default.entity_display_name" . }}' ]

# Contains long explanation of the alerted problem.
[ state_message: <tmpl_string> | default = '{{ template "victorops.default.state_message" . }}' ]

# The monitoring tool the state message is from.
[ monitoring_tool: <tmpl_string> | default = '{{ template "victorops.default.monitoring_tool" . }}' ]

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

## Web Hook 调用`<webhook_config>`

The webhook receiver allows configuring a generic receiver.

```yaml
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = true ]

# The endpoint to send HTTP POST requests to.
url: <string>

# The HTTP client's configuration.
[ http_config: <http_config> | default = global.http_config ]
```

The Alertmanager will send HTTP POST requests in the following JSON format to the configured endpoint:

```yaml
{
  "version": "4",
  "groupKey": <string>,    // key identifying the group of alerts (e.g. to deduplicate)
  "status": "<resolved|firing>",
  "receiver": <string>,
  "groupLabels": <object>,
  "commonLabels": <object>,
  "commonAnnotations": <object>,
  "externalURL": <string>,  // backlink to the Alertmanager.
  "alerts": [
    {
      "status": "<resolved|firing>",
      "labels": <object>,
      "annotations": <object>,
      "startsAt": "<rfc3339>",
      "endsAt": "<rfc3339>",
      "generatorURL": <string> // identifies the entity that caused the alert
    },
    ...
  ]
}
```

There is a list of [integrations](https://prometheus.io/docs/operating/integrations/#alertmanager-webhook-receiver) with this feature. 

## `<wechat_config>`

WeChat notifications are sent via the [WeChat API](http://admin.wechat.com/wiki/index.php?title=Customer_Service_Messages).

```
# Whether or not to notify about resolved alerts.
[ send_resolved: <boolean> | default = false ]

# The API key to use when talking to the WeChat API.
[ api_secret: <secret> | default = global.wechat_api_secret ]

# The WeChat API URL.
[ api_url: <string> | default = global.wechat_api_url ]

# The corp id for authentication.
[ corp_id: <string> | default = global.wechat_api_corp_id ]

# API request data as defined by the WeChat API.
[ message: <tmpl_string> | default = '{{ template "wechat.default.message" . }}' ]
[ agent_id: <string> | default = '{{ template "wechat.default.agent_id" . }}' ]
[ to_user: <string> | default = '{{ template "wechat.default.to_user" . }}' ]
[ to_party: <string> | default = '{{ template "wechat.default.to_party" . }}' ]
[ to_tag: <string> | default = '{{ template "wechat.default.to_tag" . }}' ]
```





####  Alert Manager inAction

[Prometheus告警简介](https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/prometheus-alert-manager-overview)



[Prometheus使用参考](https://blog.csdn.net/lijiaocn/article/details/81865120)

[Prometheus With AlertManager](https://itnext.io/prometheus-with-alertmanager-f2a1f7efabd6)

**Monitoring is incomplete without alerting.** 

##### 1.Alertmanager配置示例：

```shell
 global:
    route:
      receiver: 'email' #全局配置，默认将收到的告警事件路由给email接收器
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      routes:
      - receiver: 'auto-hpa' ＃将trigger=hpa的告警路由给auto-hpa
        match:
          trigger: hpa
    receivers:
    - name: 'email'
      email_configs:
      - to: ops@test.com
        from: monitor@test.com
        smarthost: smtpserver:port
        auth_username: "username"
        auth_identity: "username"
        auth_password: "password"
        require_tls: true       
    - name: "auto-hpa"
      webhook_configs:
      - url: 'http://YOUR_WEBHOOK_IP:PORT/hpa' #自定义webhook url地址。
        send_resolved: true
```

Alertmanager接受到相应的告警之后，会将获取到的具体metics值(此处metric name为`app_active_task_count`)和在告警规则中定义的`LABELS`信息合并为一个json数据，以POST方式发送给我我们定义好的webhook url。



##### 2. Prometheus Alert Rules 配置 

Prometheus中Alert rules的配置示例：

```yaml
    ALERT HpaTrigger
    IF app_active_task_count > 30
    FOR 30m 
    LABELS {serverity = "page",trigger="hpa",action = "scale-out",value = "{{$value}}", deployment="test", namespace = "{{$labels.namespace}}"}
    ANNOTATIONS {
      summary = "Instance {{$labels.namespace}}: scale-out",
      description = "{{$labels.namespace}} auto scale-out"
    }
```

上述规则表示应用的活动任务数持续30分钟都大于30的话，就需要创建新的pod以应对过多的任务数。但此处并不会直接触发水平Pod自动伸缩功能，prometheus根据告警规则只会生成一个告警事件，并将该事件传递给alertmanager,由alertmanager决定如何处理该告警。

## RECORDING RULES 定义

## Configuring rules

Prometheus supports two types of rules which may be configured and then evaluated at regular intervals: recording rules and [alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/). To include rules in Prometheus, create a file containing the necessary rule statements and have Prometheus load the file via the `rule_files` field in the [Prometheus configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/). Rule files use YAML.

The rule files can be reloaded at runtime by sending `SIGHUP` to the Prometheus process. The changes are only applied if all rule files are well-formatted.

## Syntax-checking rules

To quickly check whether a rule file is syntactically correct without starting a Prometheus server, install and run Prometheus's `promtool` command-line utility tool:

```
go get github.com/prometheus/prometheus/cmd/promtool
promtool check rules /path/to/example.rules.yml
```

When the file is syntactically valid, the checker prints a textual representation of the parsed rules to standard output and then exits with a `0` return status.

If there are any syntax errors or invalid input arguments, it prints an error message to standard error and exits with a `1`return status.

## Recording rules

Recording rules allow you to precompute frequently needed or computationally expensive expressions and save their result as a new set of time series. Querying the precomputed result will then often be much faster than executing the original expression every time it is needed. This is especially useful for dashboards, which need to query the same expression repeatedly every time they refresh.

Recording and alerting rules exist in a rule group. Rules within a group are run sequentially at a regular interval.

The syntax of a rule file is:

```yaml
groups:
  [ - <rule_group> ]
```

A simple example rules file would be:

```yaml
groups:
  - name: example
    rules:
    - record: job:http_inprogress_requests:sum
      expr: sum(http_inprogress_requests) by (job)
```

### `<rule_group>`

```yaml
# The name of the group. Must be unique within a file.
name: <string>

# How often rules in the group are evaluated.
[ interval: <duration> | default = global.evaluation_interval ]

rules:
  [ - <rule> ... ]
```

### `<rule>`

The syntax for recording rules is:

```yaml
# The name of the time series to output to. Must be a valid metric name.
# 时序名称：必须是一个可用的metric名称
record: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and the result recorded as a new set of
# time series with the metric name as given by 'record'.
expr: <string>

# Labels to add or overwrite before storing the result.
labels:
  [ <labelname>: <labelvalue> ]
```

The syntax for alerting rules is:

```yaml
# The name of the alert. Must be a valid metric name.
alert: <string>

# The PromQL expression to evaluate. Every evaluation cycle this is
# evaluated at the current time, and all resultant time series become
# pending/firing alerts.
expr: <string>

# Alerts are considered firing once they have been returned for this long.
# Alerts which have not yet fired for long enough are considered pending.
[ for: <duration> | default = 0s ]

# Labels to add or overwrite for each alert.
labels:
  [ <labelname>: <tmpl_string> ]

# Annotations to add to each alert.
annotations:
  [ <labelname>: <tmpl_string> ]
```



## ALERTING RULES

Alerting rules allow you to define alert conditions based on Prometheus expression language expressions and to send notifications about firing alerts to an external service. Whenever the alert expression results in one or more vector elements at a given point in time, the alert counts as active for these elements' label sets.

### Defining alerting rules

Alerting rules are configured in Prometheus in the same way as [recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/).

An example rules file with an alert would be:

```yaml
groups:
- name: example
  rules:
  - alert: HighErrorRate
    expr: job:request_latency_seconds:mean5m{job="myjob"} > 0.5
    for: 10m
    labels:
      severity: page
    annotations:
      summary: High request latency
```

The optional `for` clause causes Prometheus to wait for a certain duration between first encountering a new expression output vector element and counting an alert as firing for this element. In this case, Prometheus will check that the alert continues to be active during each evaluation for 10 minutes before firing the alert. Elements that are active, but not firing yet, are in the pending state.

The `labels` clause allows specifying a set of additional labels to be attached to the alert. Any existing conflicting labels will be overwritten. The label values can be templated.

The `annotations` clause specifies a set of informational labels that can be used to store longer additional information such as alert descriptions or runbook links. The annotation values can be templated.

#### Templating

Label and annotation values can be templated using [console templates](https://prometheus.io/docs/visualization/consoles). The `$labels` variable holds the label key/value pairs of an alert instance and `$value` holds the evaluated value of an alert instance.

```yaml
# To insert a firing element's label values:
# 触发报警metric的标签
{{ $labels.<labelname> }}
# To insert the numeric expression value of the firing element:
#表达式计算的数值
{{ $value }}
```

Examples:

```yaml
groups:
- name: example
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

  # Alert for any instance that has a median request latency >1s.
  - alert: APIHighRequestLatency
    expr: api_http_request_latencies_second{quantile="0.5"} > 1
    for: 10m
    annotations:
      summary: "High request latency on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```

### Inspecting alerts during runtime

To manually inspect which alerts are active (pending or firing), navigate to the "Alerts" tab of your Prometheus instance. This will show you the exact label sets for which each defined alert is currently active.

For pending and firing alerts, Prometheus also stores synthetic time series of the form `ALERTS{alertname="<alert name>", alertstate="pending|firing", <additional alert labels>}`. The sample value is set to `1` as long as the alert is in the indicated active (pending or firing) state, and the series is marked stale when this is no longer the case.

### Sending alert notifications

Prometheus's alerting rules are good at figuring what is broken *right now*, but they are not a fully-fledged notification solution. Another layer is needed to add summarization, notification rate limiting, silencing and alert dependencies on top of the simple alert definitions. In Prometheus's ecosystem, the [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/) takes on this role. Thus, Prometheus may be configured to periodically send information about alert states to an Alertmanager instance, which then takes care of dispatching the right notifications.
Prometheus can be [configured](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) to automatically discovered available Alertmanager instances through its service discovery integrations.