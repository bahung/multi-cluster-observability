# Introduction
This document shows how to configure observer cluster, Envoy proxy, and Thanos components.

# Configure the observer cluster
As per our design, we need to access the prometheus metrics from the centralized observer cluster across all the clusters that we need to monitor.

We need to also set up kube-prometheus-stack in the Observer cluster for to monitor the health of the Observer cluster and the components installed in the Observer cluster itself.

Create new file `observer-prod-values.yaml`
```sh
prometheus:
  thanosIngress:
    enabled: false # do not need thanos ingress
  prometheusSpec:
    externalLabels:
      cluster: observer
      stage: prod
      region: ap-northeast-1
```
We can use the default `values.yaml` and `shared-values.yaml` file in repository and install the chart:
```sh
helm upgrade -i prometheus . -n platform --values=shared-values.yaml --values=observer-prod-values.yaml
```
# Configure the envoy proxy
Because the certificate validation fails against ALB with error transport: `authentication handshake failed: x509: cannot validate certificate for xxx because it doesn't contain any IP SANs.`

Thanos underneath uses golang with gRPC protocol which fails if the certificate has missing fields.

So, we use envoy proxy as a middleware as it doesn’t perform these checks to be able to connect with the other clusters.

```sh
files:
  envoy.yaml: |-
    admin:
      access_log_path: /dev/null
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 9901
    static_resources:
      listeners:
      - name: listener_clusterA
        address:
          socket_address:
            address: 0.0.0.0
            port_value: 10000
        filter_chains:
        - filters:
          - name: envoy.filters.network.http_connection_manager
            typed_config: 
              "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
              stat_prefix: ingress_http
              access_log:
              - name: envoy.access_loggers.file
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                  path: /dev/stdout
                  log_format:
                    text_format: |
                      [%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"
                      %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %BYTES_RECEIVED% %BYTES_SENT% %DURATION%
                      %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%"
                      "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" "%UPSTREAM_TRANSPORT_FAILURE_REASON%"\n             
              - name: envoy.access_loggers.file
                typed_config:
                  "@type": type.googleapis.com/envoy.extensions.access_loggers.file.v3.FileAccessLog
                  path: /dev/stdout
              http_filters:
              - name: envoy.filters.http.router
              route_config:
                name: local_route
                virtual_hosts:
                - name: local_service
                  domains: ["*"]
                  routes:
                  - match:
                      prefix: "/"
                      grpc: {}
                    route:
                      host_rewrite_literal: clusterA.prometheus.example.net
                      cluster: service_clusterA
      clusters:
      - name: service_clusterA
        connect_timeout: 30s
        type: logical_dns
        http2_protocol_options: {}
        dns_lookup_family: V4_ONLY
        load_assignment:
          cluster_name: service_clusterA
          endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: clusterA.prometheus.example.net
                    port_value: 443
        transport_socket:
          name: envoy.transport_sockets.tls
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
            sni: clusterA.prometheus.example.net
            common_tls_context:
              alpn_protocols:
              - "h2"
```

