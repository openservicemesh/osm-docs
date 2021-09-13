---
title: "Troubleshoot OSM with osm-health"
description: "How to diagnose issues with OSM's control plane and monitored resources"
aliases: "/docs/troubleshooting/health"
type: docs
weight: 2
---

# OSM Health
osm-health is a CLI tool for troubleshooting issues with [Open Service Mesh](https://github.com/openservicemesh/osm). It can
be used to:
- Gather diagnostic information on the state of the control plane and the mesh components
- Ensure that everything is running as expected 
- Triage the state of the mesh when an issue is encountered

## Get started

1. Clone [the repo](https://github.com/openservicemesh/osm-health)

1. Checkout the tag for the [latest release](https://github.com/openservicemesh/osm-health/releases). For example:
   ```bash
   git checkout v0.0.1
   ```
   
1. From root, build the binary

    ```bash
    make build-osm-health
    ```

1. Add it to your `$PATH` to get started. For example:
    ```bash
    sudo cp ./bin/osm-health /usr/local/bin/osm-health
    ```

You can now use the CLI with:
```bash
osm-health <command>
```

## Commands
To check the status of the osm control plane, run:
```bash
osm-health control-plane status
```

osm-health can check the connectivity between two pods by running a series of diagnostic checks on the meshed namespaces and pods,
Envoy, SMI policies and core OSM control plane components. To run these checks, use:

```bash
osm-health connectivity pod-to-pod <SOURCE_POD> <DESTINATION_POD>
```

## Outcomes
A command runs a series of checks associated with that command.

Each check can return one of 4 outcomes:
1. `Pass`: indicates the check was successful and its result was as expected
1. `Fail`: indicates the check failed and returns the error that could be causing the failure. Failed checks highlight
   components that could require further investigation
1. `Info`: this is returned when the check is not generally expected to pass or fail, but rather the purpose of the
   check is to simply provide information to the user. An info check prints out general diagnostic information generated
   by the check.
   

      _For example, when SMI TrafficTarget checks are run, they may return an `info` outcome that says that permissive
      traffic policy mode is enabled, so SMI access policies do not apply. Such an outcome cannot be categorized as a
      pass or a fail outcome because it is not an unexpected behavior_
1. `Unknown`: this indicates the check could not come to a clear conclusion