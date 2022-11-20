# Introduction
This document shows how to configure observer cluster, Envoy proxy, and Thanos components.

# Configure the observer cluster
Using `helm-envoy` and `helm-thanos` charts.

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
# Configure the envoy proxy with `helm-envoy chart`
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

# Configure thanos with `helm-thanos chart`
Components of thanos:
- Thanos Querier aggregates and optionally deduplicates multiple metrics backends under single Prometheus Query endpoint.
- Thanos Store (or Store Gateway) implements the Store API on top of historical data in an object storage bucket. It acts primarily as an API gateway and therefore does not need significant amounts of local disk space.
- Thanos Compactor applies the compaction procedure of the Prometheus 2.0 storage engine to block data stored in object storage. It is also responsible for creating 5m downsampling for blocks larger than 40 hours (2d, 2w) and creating 1h downsampling for blocks larger than 10 days (2w)

Create a sample `shared-values.yaml` file with following configuration
```sh
image:
  tag: v0.18.0

sidecar:
  enabled: true
  namespace: platform

query:
  storeDNSDiscovery: true
  sidecarDNSDiscovery: true

store:
  enabled: true
  serviceAccount: "thanos-store"

bucket:
  enabled: true
  serviceAccount: "thanos-store"

compact:
  enabled: true
  serviceAccount: "thanos-store"

objstore:
  type: S3
  config:
    endpoint: s3.ap-northeast-1.amazonaws.com
    sse_config:
      type: "SSE-S3"
```

Create env specific values file for the hlem chart with `prod-values.yaml`:
```sh
objstore:
  config:
    # AWS S3 metrics bucket name
    bucket: <bucket_name>

query:
  stores: 
    - dnssrv+_clusterA._tcp.envoy
  extraArgs:
    - "--rule=dnssrv+_clusterA._tcp.envoy"
    - "--rule=dnssrv+_grpc._tcp.thanos-sidecar-grpc.platform.svc.cluster.local"
```

Here, we use `dnssrv+` scheme to detect Thanos API servers through respective DNS lookups.

`dnssrv+` scheme - With DNS SD, a domain name can be specified and it will be periodically queried to discover a list of IPs.
The domain name after this prefix will be looked up as a SRV query, and then each SRV record will be looked up as an A/AAAA query. You do not need to specify a port as the one from the query results will be used.

The second rule in the config `dnssrv+_grpc._tcp.thanos-sidecar-grpc.platform.svc.cluster.local` is required so that Thanos Querier can connect to the thanos sidecar in the local prometheus instance (observer cluster)
