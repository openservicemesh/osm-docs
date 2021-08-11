---
title: "Permissive Mode"
description: "Permissive Traffic Policy Mode"
type: docs
weight: 2
---

# Permissive Traffic Policy Mode
Permissive traffic policy mode in OSM is a mode where [SMI][1] traffic policy enforcement is bypassed. In this mode, OSM automatically discovers services that are a part of the service mesh and programs traffic policy rules on each Envoy proxy sidecar to be able to communicate with these services.

## When to use permissive traffic policy mode
Since permissive traffic policy mode bypasses [SMI][1] traffic policy enforcement, it is suitable for use when connectivity between applications within the service mesh should flow as before the applications were enrolled into the mesh. This mode is suitable in environments
where explicitly defining traffic policies for connectivity between applications is not feasible.

A common use case to enable permissive traffic policy mode is to support gradual onboarding of applications into the mesh without breaking application connectivity. Traffic routing between application services is automatically set up by OSM controller through service discovery. Wildcard traffic policies are set up on each Envoy proxy sidecar to allow traffic flow to services within the mesh.

The alternative to permissive traffic policy mode is SMI traffic policy mode, where traffic between applications is denied by default and explicit SMI traffic policies are necessary to allow application connectivity. When policy enforcement is necessary, SMI traffic policy mode must be used instead.

## Configuring permissive traffic policy mode
Permissive traffic policy mode can be enabled or disabled at the time of OSM install, or after OSM has been installed.

### Enabling permissive traffic policy mode

Enabling permissive traffic policy mode implicitly disables SMI traffic policy mode.

During OSM install using the `--set` flag:
```bash
osm install --set OpenServiceMesh.enablePermissiveTrafficPolicy=true
```

After OSM has been installed:
```bash
# Assumes OSM is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":true}}}'  --type=merge
```

### Disabling permissive traffic policy mode

Disabling permissive traffic policy mode implicitly enables SMI traffic policy mode.

During OSM install using the `--set` flag:
```bash
osm install --set OpenServiceMesh.enablePermissiveTrafficPolicy=false
```

After OSM has been installed:
```bash
# Assumes OSM is installed in the osm-system namespace
kubectl patch meshconfig osm-mesh-config -n osm-system -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'  --type=merge
```

## How it works
When permissive traffic policy mode is enabled, OSM controller discovers all services that are a part of the mesh and programs wildcard traffic routing rules on each Envoy proxy sidecar to reach every other service in the mesh. Additionally, each proxy fronting workloads that are associated with a service is configured to accept all traffic destined to the service. Depending on the application protocol of the service (HTTP, TCP, gRPC etc.), appropriate traffic routing rules are configured on the Envoy sidecar to allow all traffic for that particular type.

Refer to the [Permissive traffic policy mode demo](/docs/demos/permissive_traffic_mode) to learn more.

### Envoy configurations
In permissive mode, OSM controller programs wildcard routes for client applications to communicate with services. Following are the envoy inbound and outbound filter and route configuration snippets from the `curl` and `httpbin` sidecar proxies.

1. Outbound Envoy configuration on the `curl` client pod:

    Outbound HTTP filter chain corresponding to the `httpbin` service:
    ```json
    {
     "name": "outbound-mesh-http-filter-chain:httpbin/httpbin",
     "filter_chain_match": {
      "prefix_ranges": [
       {
        "address_prefix": "10.96.198.23",
        "prefix_len": 32
       }
      ],
      "destination_port": 14001
     },
     "filters": [
      {
       "name": "envoy.filters.network.http_connection_manager",
       "typed_config": {
        "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_man
    HttpConnectionManager",
        "stat_prefix": "http",
        "rds": {
         "config_source": {
          "ads": {},
          "resource_api_version": "V3"
         },
         "route_config_name": "rds-outbound"
        },
        "http_filters": [
         {
          "name": "envoy.filters.http.rbac"
         },
         {
          "name": "envoy.filters.http.router"
         }
        ],
        ...
       }
      }
     ]
    }
    ```

    Outbound route configuration:
    ```json
    "route_config": {
      "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
      "name": "rds-outbound",
      "virtual_hosts": [
       {
        "name": "outbound_virtual-host|httpbin.httpbin",
        "domains": [
         "httpbin.httpbin",
         "httpbin.httpbin.svc",
         "httpbin.httpbin.svc.cluster",
         "httpbin.httpbin.svc.cluster.local",
         "httpbin.httpbin:14001",
         "httpbin.httpbin.svc:14001",
         "httpbin.httpbin.svc.cluster:14001",
         "httpbin.httpbin.svc.cluster.local:14001"
        ],
        "routes": [
         {
          "match": {
           "headers": [
            {
             "name": ":method",
             "safe_regex_match": {
              "google_re2": {},
              "regex": ".*"
             }
            }
           ],
           "safe_regex": {
            "google_re2": {},
            "regex": ".*"
           }
          },
          "route": {
           "weighted_clusters": {
            "clusters": [
             {
              "name": "httpbin/httpbin",
              "weight": 100
             }
            ],
            "total_weight": 100
           }
          }
         }
        ]
       }
      ]
    }
    ```

1. Inbound Envoy configuration on the `httpbin` service pod:

    Inbound HTTP filter chain corresponding to the `httpbin` service:
    ```json
    {
     "name": "inbound-mesh-http-filter-chain:httpbin/httpbin:80",
     "filter_chain_match": {
      "destination_port": 80,
      "transport_protocol": "tls",
      "application_protocols": [
       "osm"
      ],
      "server_names": [
       "httpbin.httpbin.svc.cluster.local"
      ]
     },
     "filters": [
      {
       "name": "envoy.filters.network.http_connection_manager",
       "typed_config": {
        "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager",
        "stat_prefix": "http",
        "rds": {
         "config_source": {
          "ads": {},
          "resource_api_version": "V3"
         },
         "route_config_name": "rds-inbound"
        },
        "http_filters": [
         {
          "name": "envoy.filters.http.rbac"
         },
         {
          "name": "envoy.filters.http.router"
         }
        ]
       }
      }
     ],
     "transport_socket": {
      ...
     }
    }
    ```

    Inbound route configuration:
    ```json
    "route_config": {
      "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
      "name": "rds-inbound",
      "virtual_hosts": [
       {
        "name": "inbound_virtual-host|httpbin.httpbin",
        "domains": [
         "httpbin",
         "httpbin.httpbin",
         "httpbin.httpbin.svc",
         "httpbin.httpbin.svc.cluster",
         "httpbin.httpbin.svc.cluster.local",
         "httpbin:14001",
         "httpbin.httpbin:14001",
         "httpbin.httpbin.svc:14001",
         "httpbin.httpbin.svc.cluster:14001",
         "httpbin.httpbin.svc.cluster.local:14001"
        ],
        "routes": [
         {
          "match": {
           "headers": [
            {
             "name": ":method",
             "safe_regex_match": {
              "google_re2": {},
              "regex": ".*"
             }
            }
           ],
           "safe_regex": {
            "google_re2": {},
            "regex": ".*"
           }
          },
          "route": {
           "weighted_clusters": {
            "clusters": [
             {
              "name": "httpbin/httpbin-local",
              "weight": 100
             }
            ],
            "total_weight": 100
           }
          },
          "typed_per_filter_config": {
           "envoy.filters.http.rbac": {
            ...
           }
          }
         }
        ]
       }
      ],
      "validate_clusters": false
    }
    ```


[1]: https://smi-spec.io/
