<p align="center">
  <img src="https://cdn.rawgit.com/flant/loghouse/master/docs/logo.png" style="max-height:100%;" height="300">
</p>

___

**UPDATE (December'20): Please note loghouse is no longer being actively developed.**

Back in 2017 (when we created this project), there was no other solution to get what we (in [Flant](https://flant.com/)) desperately needed. However creating such tools is out of our primary focus, hence — while maintaining loghouse — we've been also waiting for other projects to emerge. Luckily, the Kubernetes ecosystem grows and evolves at an amazing pace. Today, we are happy to admit the existence of other solid log management solutions we can rely on. For most cases, we use [Loki](https://github.com/grafana/loki). *([Graylog](https://github.com/Graylog2/graylog2-server) and [Elasticsearch](https://github.com/elastic/elasticsearch)-based solutions — i.e. ELK, EFK — might be other options to consider.)*

It means we don't need to improve loghouse anymore. Since it's still Open Source, you are very welcome to contribute (or even [contact us](https://twitter.com/flant_com) to become a maintainer).

___

Ready to use log management solution for Kubernetes. Efficiently store big amounts of your logs (in [ClickHouse](https://github.com/yandex/ClickHouse) database), process them using a simple query language and monitor them online through web UI. Easy and quick to deploy in an already functioning Kubernetes cluster.

Status is **alpha**. However we (Flant) use it in our production Kubernetes deployments since September, 2017. Data structure might change during alpha releases, so please be careful when updating (all relevant information is published in corresponding release notes). Data structure will become stable in beta version.

Loghouse-dashboard UI demo in action (~3 Mb):

![loghouse web UI](https://cdn.rawgit.com/flant/loghouse/master/docs/web-ui-animated.gif)

# Features

* Collecting and storing logs from all Kubernetes pods efficiently:
  * [Fluentd](https://www.fluentd.org/) processes upto 10,000 log entries per second consuming 300 MB of RAM (installed at each K8s node).
  * ClickHouse makes disk space usage minimal. Examples of logs stored in our production deployments: 3.7 million entries require 1.2 GB, 300m — 13 GB, 5,35 billion — 54 GB.
* [Simple query language](docs/en/query-language.md): Easy to select entries by exact keys values or regular expressions, multiple conditions are supported with AND/OR. *Learn more in [query language docs](docs/en/query-language.md)*.
* Selecting entries based on additional containers' data available in Kubernetes API (pod's and container's names, host, namespace, labels, etc).
* Quickly & straightforward deployable to Kubernetes via Dockerfiles and Helm chart.
* Web UI made cosy and powerful:
  * Papertrail-like user experience.
  * Customizable time frames: from date to date / from now till given period (last hour, last day, etc) / seek to specific time and show logs around it.
  * Infinite scrolling of older log entries.
  * Save your queries to use in future.
  * Basic permissions (limiting entries shown for users by specifying Kubernetes namespaces).
  * Exporting current query's results to CSV (more formats will be supported).
* fluentd monitoring via Prometheus with Grafana dashboards for ClickHouse and fluentd.


# Installation

To install loghouse, you need to use [Helm](https://github.com/kubernetes/helm). Minimal kubernetes cluster version is **>=1.9**. Also, it is considered that [cert-manager](https://github.com/jetstack/cert-manager) is already installed in your cluster.

The whole process is as simple as these two steps:

1. Add loghouse charts:
```
# helm repo add loghouse https://flant.github.io/loghouse/charts/
```

2. Install a chart.

2.1. Easy way:

```
# helm fetch loghouse/loghouse --untar
# vim loghouse/values.yaml
# helm install --namespace loghouse -n loghouse loghouse
```

Note: use `--timeout 1200` flag in case slow image pulling.

2.2. Using specific parameters *(check variables in chart's [values.yaml](charts/loghouse/values.yaml) — not documented yet)*:

```
# helm install -n loghouse loghouse/loghouse --set 'param=value' ...
```

Web UI (loghouse-dashboard) will be reachable via address specified in values.yaml config as ```loghouse_host```. You'll be prompted by basic authorization generated via htpasswd and configured in ```auth``` parameter of your values.yaml.

> To clean old logs in cron, you can use a script in this [issue](https://github.com/flant/loghouse/issues/42).

# Architecture

![loghouse architecture](docs/architecture.png)

A pod with fluentd collecting logs will be installed on each node of your Kubernetes cluster. Technically, it is implemented by means of DaemonSet having tolerations for all possible taints, so it includes all the nodes available. Logs directories from all hosts are mounted to fluentd pods and watched by fluentd. [Kubernetes_metadata](https://github.com/fabric8io/fluent-plugin-kubernetes_metadata_filter) filter is applied for all the Docker containers' logs to get additional information about containers via Kubernetes API. Another filter, [record_modifier](https://github.com/repeatedly/fluent-plugin-record-modifier), is then used to "prepare" all the data we have. And the last step is passing this data to fluentd output plugin executing [clickhouse-client](https://clickhouse.yandex/docs/en/interfaces/cli.html) CLI tool to insert new entries into ClickHouse DBMS.

**Note on logs format**: If log entry is in JSON, it will be formatted according to its values' types, i.e. each field will be stored in corresponding table: string_fields, number_fields, boolean_fields, null_fields or labels (the last one is used for containers labels to make further filtering and lookups easy). ClickHouse built-in functions will be used to process these data types. If log entry isn't in JSON, it will be stored in string_fields table.

By default, ClickHouse DBMS is deployed as a single instance via StatefulSet which brings this instance to a random K8s node (this behaviour can be changed by using nodeSelector and tolerations to choose a specific node). ClickHouse stores its data in hostPath volume or Persistent Volumes Claim (PVC) created with any storageClass you prefer. You can find other deployment options for ClickHouse (i.e. using an external ClickHouse instance) [here](docs/en/schemas/README.md).

Web UI ([screenshot](docs/loghouse_interface.png)) is composed of two components:

* **frontend** — nginx with basic authorization. This authorization is used to limit user's access with logs from given Kubernetes namespaces only;
* **backend** — Ruby application displaying logs from ClickHouse.

ClickHouse and Fluentd components have optional Prometheus exporters. You can find our default dashboards for Grafana [here](docs/en/grafana).

# Upgrading

In **v0.3.0**, Helm chart for loghouse has been rewritten. All API versions as well as Helm hook policy have been updated. To upgrade your loghouse installation to v0.3.0, you have to remove conflicting objects:

```
kubectl -n loghouse delete jobs,ing --all
```

**Warning!** Database schema has been also changed. Please, prepare a backup of ClickHouse data before upgrading. Note that the migration task will be automatically started at the end of the upgrade process.

# Roadmap

We're going to add logs filtering, another deployment options (having ClickHouse instances on each K8s node), migrate frontend to AngularJS and backend to Golang, add command-line interface (CLI) and much more.

More details will be available in our [issues](https://github.com/flant/loghouse/issues).
