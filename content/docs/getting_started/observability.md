---
title: "Observability"
description: "Use OSM's observability integrations to inspect the traffic between the bookstore applications"
type: docs
weight: 5
---

# Observability

On `osm install`, a Prometheus and/or Grafana instance can be automatically provisioned with the default OSM configuration.
```bash
 osm install --set=osm.deployPrometheus=true \
             --set=osm.deployGrafana=true
```
More information on observability can be found in the [Observability Guide](/docs/guides/observability).

## Prometheus

When configured with the `--set=osm.deployPrometheus=true` flag, OSM installation will deploy a Prometheus instance to scrape the sidecar and OSM control plane's metrics endpoints. The [scraping configuration file](https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/charts/osm/templates/prometheus-configmap.yaml) defines the default Prometheus behavior and the set of metrics collected by OSM.

## Grafana

OSM can be configured to deploy a [Grafana](https://grafana.com/grafana/) instance using the `--set=osm.deployGrafana=true` flag in `osm install`. OSM provides pre-configured dashboards that are documented in the [OSM Grafana dashboards](/docs/guides/observability/metrics/#osm-grafana-dashboards) section of the Observability Guide.

## Inspect Dashboards

The OSM Grafana dashboards can be viewed with the following command:

```bash
osm dashboard
```

> Note: If you still have the additional terminal still running the `./scripts/port-forward-all.sh` script, go ahead and `CTRL+C` to terminate the port forwarding. The `osm dashboard` port redirection will not work simultaneously with the port forwarding script still running.

Navigate to http://localhost:3000 to access the Grafana dashboards. The default user name is `admin` and the default password is `admin`. On the Grafana homepage click on the **Home** icon, you will see a folder containing dashboards for both OSM Control Plane and OSM Data Plane.
