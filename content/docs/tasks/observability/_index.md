---
title: "Configuring Observability"
description: "OSM's observability stack includes Prometheus for metrics collection, Grafana for metrics visualization, Jaeger for tracing and Fluent Bit for log forwarding."
aliases: "/docs/troubleshooting/observability/"
type: docs
---


OSM's observability stack includes Prometheus for metrics collection, Grafana for metrics visualization, Jaeger for tracing and Fluent Bit for log forwarding to a user-defined endpoint. These are disabled by default but may be enabled during OSM installation with flags `--deploy-prometheus`, `--deploy-grafana`, `--deploy-jaeger` and `--enable-fluentbit`.

## Table of Contents
- [Metrics - Prometheus and Grafana](/docs/tasks/observability/metrics)
- [Tracing - Jaeger](/docs/tasks/observability/tracing)
- [Log forwarding - Fluent Bit](/docs/tasks/observability/logs)
