Elasticsearch
=============

The Elasticsearch chart in openstack-helm-infra provides a distributed data
store to index and analyze logs generated from the OpenStack-Helm services.
The chart contains templates for:

- Elasticsearch client nodes
- Elasticsearch data nodes
- Elasticsearch master nodes
- An Elasticsearch exporter for providing cluster metrics to Prometheus
- A cronjob for Elastic Curator to manage data indices

Elasticsearch Service Configuration
-----------------------------------

The Elasticsearch service configuration file can be modified with a combination
of pod environment variables and entries in the values.yaml file.  Elasticsearch
does not require much configuration out of the box, and the default values for
these configuration settings are meant to provide a highly available cluster by
default.

The vital entries in this configuration file are:

- path.data:  The path at which to store the indexed data
- path.repo:  The location of any snapshot repositories to backup indexes
- bootstrap.memory_lock:  Ensures none of the JVM is swapped to disk
- discovery.zen.minimum_master_nodes:  Minimum required masters for the cluster

The bootstrap.memory_lock entry ensures none of the JVM will be swapped to disk
during execution, and setting this value to false will negatively affect the
health of your Elasticsearch nodes.  The discovery.zen.minimum_master_nodes flag
registers the minimum number of masters required for your Elasticsearch cluster
to register as healthy and functional.

To read more about Elasticsearch's configuration file, please see the official
documentation_.

.. _documentation: https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html

Elastic Curator
---------------

The Elasticsearch chart contains a cronjob to run Elastic Curator at specified
intervals to manage the lifecycle of your indices.  Curator can perform:

- Take and send a snapshot of your indexes to a specified snapshot repository
- Delete indexes older than a specified length of time
- Restore indexes with previous index snapshots
- Reindex an index into a new or preexisting index

The full list of supported Curator actions can be found in the actions_ section of
the official Curator documentation.  The list of options available for those
actions can be found in the options_ section of the Curator documentation.

.. _actions: https://www.elastic.co/guide/en/elasticsearch/client/curator/current/actions.html
.. _options: https://www.elastic.co/guide/en/elasticsearch/client/curator/current/options.html

Curator's configuration is handled via entries in Elasticsearch's values.yaml
file and can be overridden wholesale to achieve your index lifecycle management
needs.  Please note that any unused field should be left blank, as an entry of
"None" will result in an exception, as Curator will read it as a Python NoneType
insead of a value of None.

The section for Curator's service configuration can be found at:

::

    conf:
      curator:
        config:
          client:
            hosts:
              - elasticsearch-logging
            port: 9200
            url_prefix:
            use_ssl: False
            certificate:
            client_cert:
            client_key:
            ssl_no_validate: False
            http_auth:
            timeout: 30
            master_only: False
          logging:
            loglevel: INFO
            logfile:
            logformat: default
            blacklist: ['elasticsearch', 'urllib3']

Curator's actions are configured in the following section:

::

    conf:
      curator:
        action_file:
          actions:
            1:
              action: delete_indices
              description: "Clean up ES by deleting old indices"
              options:
                timeout_override:
                continue_if_exception: False
                ignore_empty_list: True
                disable_action: True
              filters:
              - filtertype: age
                source: name
                direction: older
                timestring: '%Y.%m.%d'
                unit: days
                unit_count: 30
                field:
                stats_result:
                epoch:
                exclude: False

The Elasticsearch chart contains example actions for deleting and snapshotting
indexes older 30 days.  Please note these actions are provided as a reference
and are disabled by default to avoid any unexpected behavior against your
indexes.

Elasticsearch Exporter
----------------------

The Elasticsearch chart contains templates for an exporter to provide metrics
for Prometheus.  These metrics provide insight into the performance and overall
health of your Elasticsearch cluster.  Please note the exporter is disabled by
default, and must be overriden during chart installation.

The flags for the Elasticsearch exporter can be found under the manifests key,
and should be enabled via overrides during installation:

::

    manifests:
      configmap_bin: true
      configmap_etc: true
      cron_curator: true
      deployment_client: true
      deployment_master: true
      helm_tests: true
      job_image_repo_sync: true
      job_snapshot_repository: false
      monitoring:
        prometheus:
          configmap_bin: true
          deployment_exporter: true
          service_exporter: true
      pvc_snapshots: false
      secret_admin: true
      service_data: true
      service_discovery: true
      service_logging: true
      statefulset_data: true


The Elasticsearch exporter uses the same service annotations as the other
exporters, and no additional configuration is required for Prometheus to target
the Elasticsearch exporter for scraping.  The Elasticsearch exporter is
configured with command line flags, and the flags' default values can be found
under the following key in the values.yaml file:

::

    conf:
      prometheus_elasticsearch_exporter:
        es:
          all: true
          timeout: 20s

The configuration keys configure the following behaviors:

- es.all:  Gather information from all nodes, not just the connecting node
- es.timeout:  Timeout for metrics queries

More information about the Elasticsearch exporter can be found on the exporter's
GitHub_ page.

.. _GitHub: https://github.com/justwatchcom/elasticsearch_exporter


Snapshot Repositories
---------------------

Before Curator can store snapshots in a specified repository, Elasticsearch must
register the configured repository.  To achieve this, the Elasticsearch chart
contains a job for registering a snapshot repository.  This job is disabled by
default as the curator actions for snapshots are disabled by default.  To
enable the snapshot job, the manifests.job_snapshot_repository key must be set
to true and the desired repository must be configured via the following
values:

- conf.elasticsearch.repository.enabled: Enable snapshot repositories
- conf.elasticsearch.repository.name: Name of the repository
- conf.elasticsearch.repository.type: Type of repository defined
- conf.elasticsearch.repository.location: Location of the snapshot repository

More information about Elasticsearch repositories can be found in the official
Elasticsearch snapshot_ documentation:

.. _snapshot: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-snapshots.html#_repositories

The Elasticsearch chart includes an optional template for a PVC to provide a
file system repository backed by NFS.  The values for the repository PVC
can be found at:

::

    storage:
      filesystem_repository:
        enabled: false
        pvc:
          name: pvc-snapshots
          access_mode: ReadWriteMany
        requests:
          storage: 5Gi
        storage_class: general

Please note this PVC and its template are disabled by default, and this
repository is set as the default target for the repository bootstrap job.  If
you wish to enable this repository, set the storage.filesystem_repository.enabled
flag and manifests.pvc_snapshots flag to true.  This will result in volume
mounts for your Elasticsearch master and data pods that are backed by the
PVC defined above.
