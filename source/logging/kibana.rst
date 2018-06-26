Kibana
======

The Kibana chart in OpenStack-Helm Infra provides visualization for logs indexed
into Elasticsearch.  These visualizations provide the means to view logs captured
from services deployed in cluster and targeted for collection by Fluentbit.

Configuration
-------------

Kibana's configuration is driven by the chart's values.yaml file.  The configuration
options are found under the following keys:

::

    conf:
      elasticsearch:
        pingTimeout: 1500
        preserveHost: true
        requestTimeout: 30000
        shardTimeout: 0
        startupTimeout: 5000
      il8n:
        defaultLocale: en
      kibana:
        defaultAppId: discover
        index: .kibana
      logging:
        quiet: false
        silent: false
        verbose: false
      ops:
        interval: 5000
      server:
        host: 0.0.0.0
        maxPayloadBytes: 1048576
        port: 5601
        ssl:
          enabled: false

The case of the sub-keys is important as these values are injected into
Kibana's configuration configmap with the toYaml function.  More information on
the configuration options and available settings can be found in the official
Kibana documentation_.

.. _documentation: https://www.elastic.co/guide/en/kibana/current/settings.html

Installation
------------

At the moment, Kibana should be deployed with a single replica until sessions are
supported for the chart.  The chart can be deployed with the following:

.. code_block: bash

helm install --namespace=<namespace> local/kibana --name=kibana \
  --set pod.replicas.kibana=1

Setting Time Field
------------------

For Kibana to successfully read the logs from Elasticsearch's indexes, the time
field will need to be manually set after Kibana has successfully deployed.  Upon
visiting the Kibana dashboard for the first time, a prompt will appear to choose the
time field with a drop down menu.  The default time field for Elasticsearch indexes
is '@timestamp'.  Once this field is selected, the default view for querying log entries
can be found by selecting the "Discover"
