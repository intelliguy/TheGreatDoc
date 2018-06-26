Prometheus
==========

The Prometheus chart in openstack-helm-infra provides a time series database and
a strong querying language for monitoring various components of OpenStack-Helm.
Prometheus gathers metrics by scraping defined service endpoints or pods at
specified intervals and indexing them in the underlying time series database.

Prometheus Service configuration
--------------------------------

The Prometheus service is configured via command line flags set during runtime.
These flags include: setting the configuration file, setting log levels, setting
characteristics of the time series database, and enabling the web admin API for
snapshot support.  These settings can be configured via the values tree at:

::

    conf:
      prometheus:
        storage:
        log:
        query:
        web_admin_api:

The Prometheus configuration file contains the definitions for scrape targets
and the location of the rules files for triggering alerts on scraped metrics.
The configuration file is defined in the values file, and can be found at:

::

    conf:
      prometheus:
        scrape_configs: |

By defining the configuration via the values file, an operator can override all
configuration components of the Prometheus deployment at runtime.

Kubernetes Endpoint Configuration
---------------------------------

The Prometheus chart in openstack-helm-infra uses the built-in service discovery
mechanisms for Kubernetes endpoints and pods to automatically configure scrape
targets.  Functions added to helm-toolkit allows configuration of these targets
via annotations that can be applied to any service or pod that exposes metrics
for Prometheus, whether a service for an application-specific exporter or an
application that provides a metrics endpoint via its service. The values in
these functions correspond to entries in the monitoring tree under the
prometheus key in a chart's values.yaml file.


The functions definitions are below:

::

    {{- define "helm-toolkit.snippets.prometheus_service_annotations" -}}
    {{- $config := index . 0 -}}
    {{- if $config.scrape }}
    prometheus.io/scrape: {{ $config.scrape | quote }}
    {{- end }}
    {{- if $config.scheme }}
    prometheus.io/scheme: {{ $config.scheme | quote }}
    {{- end }}
    {{- if $config.path }}
    prometheus.io/path: {{ $config.path | quote }}
    {{- end }}
    {{- if $config.port }}
    prometheus.io/port: {{ $config.port | quote }}
    {{- end }}
    {{- end -}}

::

    {{- define "helm-toolkit.snippets.prometheus_pod_annotations" -}}
    {{- $config := index . 0 -}}
    {{- if $config.scrape }}
    prometheus.io/scrape: {{ $config.scrape | quote }}
    {{- end }}
    {{- if $config.path }}
    prometheus.io/path: {{ $config.path | quote }}
    {{- end }}
    {{- if $config.port }}
    prometheus.io/port: {{ $config.port | quote }}
    {{- end }}
    {{- end -}}

These functions render the following annotations:

- prometheus.io/scrape:  Must be set to true for Prometheus to scrape target
- prometheus.io/scheme:  Overrides scheme used to scrape target if not http
- prometheus.io/path:    Overrides path used to scrape target metrics if not /metrics
- prometheus.io/port:    Overrides port to scrape metrics on if not service's default port

Each chart that can be targeted for monitoring by Prometheus has a prometheus
section under a monitoring tree in the chart's values.yaml, and Prometheus
monitoring is disabled by default for those services.  Example values for the
required entries can be found in the following monitoring configuration for the
prometheus-kube-state-metrics chart:

::

    monitoring:
      prometheus:
        enabled: false
        node_exporter:
          scrape: true

If the prometheus.enabled key is set to true, the annotations are set on the
targeted service or pod as the condition for applying the annotations evaluates
to true.  For example:

::

    {{- $prometheus_annotations := $envAll.Values.monitoring.prometheus.node_exporter }}
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: {{ tuple "node_metrics" "internal" . | include "helm-toolkit.endpoints.hostname_short_endpoint_lookup" }}
    labels:
    {{ tuple $envAll "node_exporter" "metrics" | include "helm-toolkit.snippets.kubernetes_metadata_labels" | indent 4 }}
    annotations:
    {{- if .Values.monitoring.prometheus.enabled }}
    {{ tuple $prometheus_annotations | include "helm-toolkit.snippets.prometheus_service_annotations" | indent 4 }}
    {{- end }}

Kubelet, API Server, and cAdvisor
---------------------------------

The Prometheus chart includes scrape target configurations for the kubelet, the
Kubernetes API servers, and cAdvisor.  These targets are configured based on
a kubeadm deployed Kubernetes cluster, as OpenStack-Helm uses kubeadm to deploy
Kubernetes in the gates.  These configurations may need to change based on your
chosen method of deployment.  Please note the cAdvisor metrics will not be
captured if the kubelet was starte
