---
title: Configuring Envoy as an edge proxy - through istio
description: How we implemented Envoy edge proxy best practices in istio-ingressgateway.
date: 2024-11-25 00:00:00+0000
image: envoy-logo.svg
categories:
    - Technology
tags:
    - kubernetes
    - istio
---

- [Introduction](#introduction)
- [Configuring overload manager and global connection limits using a custom Envoy bootstrap](#custom-bootstrap)
- [Configuring buffer sizes and connection timeouts via EnvoyFilter](#envoyfilter)
- [Appendixes](#appendixes)
  - [Appendix A - Displaying currently active Envoy edge configuration settings](#appendix-get-envoy-config)
  - [Appendix B - Overload manager metrics](#appendix-overload-manager-metrics)
  - [Appendix C - istio installation and configuration at Signicat](#appendix-istio-at-signicat)
- [Outro](#outro)

*This is a cross-post of a blog post also published on the [Signicat Blog]()*

<a id="introduction"></a>
# Introduction

If you are using istio as a Service Mesh for your Kubernetes clusters, chances are you are also using `istio-ingressgateway` to handle incoming traffic from the Internet.

Envoy however, which istio relies on, is not tuned for running at the edge by default:

> Envoy is a production-ready edge proxy, however, the default settings are tailored for the service mesh use case, and some values need to be adjusted when using Envoy as an edge proxy.
> <br>— <cite>[envoyproxy.io/docs](https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge)</cite>

The [Envoy edge proxy best practices](https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge) document outlines specific recommended configuration parameters for running envoy (and thus `istio-ingressgateway`) on the edge.

It's not immediately obvious how you would propagate these configurations through the regular istio installation and configuration procedures. There is an [open feature request on GitHub](https://github.com/istio/istio/issues/24715) asking for the ability to configure `istio-ingressgateway` according to best practices.

Here I'll show you at least one way of getting these settings deployed.

---

We need two different approaches to achieve our goals. A [custom bootstrap configuration](https://github.com/istio/istio/tree/master/samples/custom-bootstrap) and an [EnvoyFilter](https://istio.io/latest/docs/reference/config/networking/envoy-filter/).

<a id="custom-bootstrap"></a>
# Configuring overload manager and global connection limits using a custom Envoy bootstrap

To enable/configure [Envoy overload manager](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/overload_manager) and global connection limits we first create our config file:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-envoy-custom-bootstrap-config
  namespace: istio-system
data:
  custom_bootstrap.yaml: |
    # Untrusted downstreams:
    overload_manager:
      refresh_interval: 0.25s
      resource_monitors:
      - name: "envoy.resource_monitors.fixed_heap"
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.resource_monitors.fixed_heap.v3.FixedHeapConfig
          max_heap_size_bytes: 350000000 # 350000000=350MB
      - name: "envoy.resource_monitors.global_downstream_max_connections"
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.resource_monitors.downstream_connections.v3.DownstreamConnectionsConfig
          max_active_downstream_connections: 25000
      actions:
      # Possible actions: https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager#overload-actions
      - name: "envoy.overload_actions.shrink_heap"
        triggers:
        - name: "envoy.resource_monitors.fixed_heap"
          threshold:
            value: 0.9
      - name: "envoy.overload_actions.stop_accepting_requests"
        triggers:
        - name: "envoy.resource_monitors.fixed_heap"
          threshold:
            value: 0.95
      # Additional settings from https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager
      - name: "envoy.overload_actions.disable_http_keepalive"
        triggers:
          - name: "envoy.resource_monitors.fixed_heap"
            threshold:
              value: 0.95
      # From https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager#reducing-timeouts
      - name: "envoy.overload_actions.reduce_timeouts"
        triggers:
          - name: "envoy.resource_monitors.fixed_heap"
            scaled:
              scaling_threshold: 0.85
              saturation_threshold: 0.95
        typed_config:
          "@type": type.googleapis.com/envoy.config.overload.v3.ScaleTimersOverloadActionConfig
          timer_scale_factors:
            - timer: HTTP_DOWNSTREAM_CONNECTION_IDLE
              min_timeout: 2s
      # https://www.envoyproxy.io/docs/envoy/latest/configuration/operations/overload_manager/overload_manager#load-shed-points
      loadshed_points:
        - name: "envoy.load_shed_points.tcp_listener_accept"
          triggers:
            - name: "envoy.resource_monitors.fixed_heap"
              threshold:
                value: 0.95
    # From https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge#best-practices-edge / https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/runtime#config-listeners-runtime
    # Also mentioned in https://istio.io/latest/news/security/istio-security-2020-007/#mitigation with much higher limits
    # Here we configure one limit for the "regular" public listener on port 8443 and a separate global limit that is higher to
    # avoid starving connections for admin and metrics and probes
    layered_runtime:
      layers:
      - name: static_layer_0
        static_layer:
          envoy:
            resource_limits:
              listener:
                0.0.0.0_8443:
                  connection_limit: 10000
```

Let's save it to `istio-envoy-custom-bootstrap-config.yaml`.

**There is a field here you MUST adjust to your environment.** That is the `max_heap_size_bytes` which we set to about 90% of the configured K8s memory limit.

What this does is inform the overload manager of how much memory it has available, and is used for evaluating percentage of current usage compared to what it thinks it has available, that again triggers overload actions at certain thresholds.

You may also have to adjust the second to last line (`0.0.0.0_8443`) in case your public listener is named something else.

Now we install it in the cluster:

```bash
kubectl apply -n istio-system -f istio-envoy-custom-bootstrap-config.yaml
```

Then we can use an [overlay](https://istio.io/latest/docs/setup/additional-setup/customize-installation/) to modify the Deployment that `istioctl` produces, before `istioctl` actually installs it in the cluster. This is the path and contents of what you need to add to your existing IstioOperator that you feed to `istioctl`:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:        
    ingressGateways:
      - name: istio-ingressgateway
        k8s:
          overlays:
            - kind: Deployment
              name: istio-ingressgateway
              patches:
                - path: spec.template.spec.containers.[name:istio-proxy].env[-1]
                  value:
                    name: ISTIO_BOOTSTRAP_OVERRIDE
                    value: /etc/istio/custom-bootstrap/custom_bootstrap.yaml
                - path: spec.template.spec.containers.[name:istio-proxy].volumeMounts[-1]
                  value:
                    mountPath: /etc/istio/custom-bootstrap
                    name: custom-bootstrap-volume
                    readOnly: true
                - path: spec.template.spec.volumes[-1]
                  value:
                    configMap:
                      name: istio-envoy-custom-bootstrap-config
                      defaultMode: 420
                      optional: false
                    name: custom-bootstrap-volume
```

> Huge shoutout to our emminent [Csaba](https://www.linkedin.com/in/csaba-k%C3%A1rp%C3%A1ti-4a229086/) which helped me out actually getting the overlay above work.

<a id="envoyfilter"></a>
# Configuring buffer sizes and connection timeouts via EnvoyFilter

We set the rest of the recommended configurations via an EnvoyFilter.

Create a file for it, named `listener-filters-edge.yaml` for example:

```yaml
# Based on recommendations for edge deployments with untrusted downstreams:
# - https://www.envoyproxy.io/docs/envoy/latest/configuration/best_practices/edge#best-practices-edge
# - https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts#faq-configuration-timeouts

apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: listener-filters-edge
spec:
  workloadSelector:
    labels:
      istio: ingressgateway
  configPatches:
    - applyTo: LISTENER
      match:
        context: GATEWAY
      patch:
        operation: MERGE
        value:
          per_connection_buffer_limit_bytes: 32768 # Doc examples 32 KiB # Default 1MB
    - applyTo: NETWORK_FILTER
      match:
        context: GATEWAY
        listener:
          filterChain:
            filter:
              name: "envoy.filters.network.http_connection_manager"
      patch:
        operation: MERGE
        value:
          name: "envoy.filters.network.http_connection_manager"
          typed_config:
            "@type": "type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager"
            # https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-request-headers-timeout
            request_headers_timeout: 10s                         # Default no timeout
            # https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#envoy-v3-api-msg-config-core-v3-http1protocoloptions
            common_http_protocol_options:
              max_connection_duration: 60s                        # Default no timeout
              idle_timeout: 900s                                  # Default 1 hour. Doc example 900s
              headers_with_underscores_action: REJECT_REQUEST
            # https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/protocol.proto#config-core-v3-http2protocoloptions
            http2_protocol_options:
              max_concurrent_streams: 100                         # Default 2147483647
              initial_stream_window_size: 65536                   # Doc examples 64 KiB - Default 268435456 (256 * 1024 * 1024)
              initial_connection_window_size: 1048576             # Doc examples 1 MiB - Same default as initial_stream_window_size
            # https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html#extensions-filters-network-http-connection-manager-v3-httpconnectionmanager
            stream_idle_timeout: 300s               # Default 5 mins. Must be disabled for long-lived and streaming requests
            request_timeout: 300s                   # Default no timeout. Must be disabled for long-lived and streaming requests
            use_remote_address: true
            normalize_path: true
            merge_slashes: true
            path_with_escaped_slashes_action: UNESCAPE_AND_REDIRECT

```

And install it like usual:

```bash
kubectl apply -n istio-system -f listener-filters-edge.yaml
```

<a id="appendixes"></a>
# Appendixes

<a id="appendix-get-envoy-config"></a>
## Appendix A - Displaying currently active Envoy edge configuration settings

Here is a handy script for printing the currently active settings to console. Useful for verifying the changes have actually made it all the way to Envoy.

```bash
CONFIG_FILE=igw_config.json

echo "Looking for pods labeled istio=ingressgateway"
ISTIO_INGRESSGATEWAY_POD=$(kubectl get pods -n istio-system -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}')

echo "Using $ISTIO_INGRESSGATEWAY_POD and dumping configuration to $CONFIG_FILE"

kubectl exec -n istio-system $ISTIO_INGRESSGATEWAY_POD curl http://localhost:15000/config_dump > $CONFIG_FILE

echo "Custom bootstrap configuration: "

printf "bootstrap.overload_manager.refresh_interval: "
cat $CONFIG_FILE | jq -r '.configs[0].bootstrap.overload_manager.refresh_interval'

printf "max_active_downstream_connections: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.resource_monitors[] | select(.name == "envoy.resource_monitors.global_downstream_max_connections") | .typed_config.max_active_downstream_connections'

printf "max_heap_size_bytes: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.resource_monitors[] | select(.name == "envoy.resource_monitors.fixed_heap") | .typed_config.max_heap_size_bytes'

printf "overload_actions.shrink_heap: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.shrink_heap") | .triggers[0].threshold.value'

printf "overload_actions.stop_accepting_requests: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.stop_accepting_requests") | .triggers[0].threshold.value'

printf "overload_actions.disable_http_keepalive: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.disable_http_keepalive") | .triggers[0].threshold.value'

printf "overload_actions.reduce_timeouts scaling_threshold: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.reduce_timeouts") | .triggers[0].scaled.scaling_threshold'

printf "overload_actions.reduce_timeouts saturation_threshold: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.reduce_timeouts") | .triggers[0].scaled.saturation_threshold'

printf "overload_actions.reduce_timeouts timer_scale_factors timer: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.reduce_timeouts") | .typed_config.timer_scale_factors[0].timer'

printf "overload_actions.reduce_timeouts timer_scale_factors min_timeout: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.actions[] | select(.name == "envoy.overload_actions.reduce_timeouts") | .typed_config.timer_scale_factors[0].min_timeout'

printf "load_shed_points.tcp_listener_accept: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.overload_manager.loadshed_points[] | select(.name == "envoy.load_shed_points.tcp_listener_accept") | .triggers[0].threshold.value'

printf "resource_limits.0.0.0.0_8443.connection_limit: "
cat $CONFIG_FILE | jq '.configs[0].bootstrap.layered_runtime.layers[] | select(.name == "static_layer_0") | .static_layer.envoy.resource_limits.listener."0.0.0.0_8443".connection_limit'

echo "EnvoyFilter configuration: "

printf "per_connection_buffer_limit_bytes: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.per_connection_buffer_limit_bytes'

printf "http2_protocol_options.max_concurrent_streams: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.http2_protocol_options.max_concurrent_streams'

printf "http2_protocol_options.initial_stream_window_size: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.http2_protocol_options.initial_stream_window_size'

printf "http2_protocol_options.initial_connection_window_size: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.http2_protocol_options.initial_connection_window_size'

printf "stream_idle_timeout: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.stream_idle_timeout'

printf "request_timeout: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.request_timeout'

printf "common_http_protocol_options.idle_timeout: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.common_http_protocol_options.idle_timeout'

printf "common_http_protocol_options.max_connection_duration: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.common_http_protocol_options.max_connection_duration'

printf "request_headers_timeout: "
cat $CONFIG_FILE | jq '.configs[2].dynamic_listeners[] | select(.name == "0.0.0.0_8443") | .active_state.listener.filter_chains[0].filters[0].typed_config.request_headers_timeout'
```

Save it to `get-edge-config-values.sh` and run it with for example `bash get-edge-config-values.sh`.

<a id="appendix-overload-manager-metrics"></a>
## Appendix B - Overload manager metrics

Envoy can also export metrics related to overload manager, they can be enabled by adding `overload` to approximately this location in the `IstioOperator` spec:

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    ingressGateways:
      - name: istio-ingressgateway
        k8s:
          podAnnotations:
            proxy.istio.io/config: |-
              proxyStatsMatcher:
                inclusionPrefixes:
                - "overload"
```

<a id="appendix-istio-at-signicat"></a>
## Appendix C - istio installation and configuration at Signicat

At Signicat we use [helm](https://helm.sh/) to template all resources going in to our istio installation. That includes resource types like:
- IstioOperator
- EnvoyFilters
- ConfigMaps
- Gateways
- Sidecars
and so on.

We don't use `helm` to install anything. Only to generate a set of manifests that is then used as inputs to the appropriate tools, like feeding generated `IstioOperator` manifests to `istioctl` and `EnvoyFilter` manifests to `kubectl`.

This works fairly well and allows us to have the same set of base manifests with adjustable values and feature sets per environment and using standard `helm` that most platform engineers are already familiar with.

We also have a couple of other tricks to enable us to have zero-downtime blue-green upgrades to `istio-ingressgateway` that we may cover in a future post.

<a id="outro"></a>
# Outro

I hope this was helpful on your journey towards scale, resilience and reliability.

The next chapter in this saga would be once we complete extensive load testing with the new configurations compared to the defaults as well as trying to find optimal values. And associated Grafana dashboards are always nice!
