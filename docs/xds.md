
# xDS protocol support

- [Protocol buffers map for xDS v3 API](#protocol-buffers-map-for-xds-v3-api)
- [xDS bootstrap file](#xds-bootstrap-file)

The gRPC team believe that Envoy proxy (actually, any data plane) is not the only solution for service mesh. By support xDS protocol gRPC can take the role of Envoy proxy. In general gRPC wants to build a proxy-less service mesh without data plane.  See [xDS Support in gRPC - Mark D. Roth](https://www.youtube.com/watch?v=IbcJ8kNmsrE) and [Traffic Director and gRPC—proxyless services for your service mesh](https://cloud.google.com/blog/products/networking/traffic-director-supports-proxyless-grpc).

From the view of data plane API, envoy proxy is a client. gRPC is another different client, while gRPC only supports partial capability of Envoy proxy. Although they are different client with different design goal, they may share the same management server (control plane) and the same data plane API.  

The following is the design document for xDS protocol support. It's a good start point to understand the code. While it's not easy to understand these documents if you are not familiar with Envoy proxy. It took me several weeks to read the [Envoy document](https://www.envoyproxy.io/docs/envoy/latest/about_docs) and [xDS REST and gRPC protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol) before the following documents.

- [xDS-Based Global Load Balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)
- [Load Balancing Policy Configuration](https://github.com/grpc/proposal/blob/master/A24-lb-policy-config.md)
- [gRPC xDS traffic splitting and routing](https://github.com/grpc/proposal/blob/master/A28-xds-traffic-splitting-and-routing.md)
- [xDS v3 Support](https://github.com/grpc/proposal/blob/master/A30-xds-v3.md)
- [gRPC xDS Timeout Support and Config Selector Design](https://github.com/grpc/proposal/blob/master/A31-xds-timeout-support-and-config-selector.md)
- [gRPC xDS circuit breaking](https://github.com/grpc/proposal/blob/master/A32-xds-circuit-breaking.md)

There are [four variants of the xDS Transport Protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol#four-variants). gRPC only supports the Aggregate Discovery Service (ADS) variant of xDS. Start from 2021, xDS v3 is the main API version supported by gRPC.

"In the future, we may add support for the incremental ADS variant of xDS. However, we have no plans to support any non-aggregated variants of xDS, nor do we plan to support REST or filesystem subscription."

[RouteConfiguration](https://github.com/envoyproxy/envoy/blob/9e83625b16851cdc7e4b0a4483b0ce07c33ba76b/api/envoy/api/v2/route.proto#L24) in Envoy is different thing from [ServiceConfig](https://github.com/grpc/grpc-proto/blob/master/grpc/service_config/service_config.proto) in gRPC. While gRPC intends to populate `ServiceConfig` with the data from `RouteConfiguration`.  

## Protocol buffers map for xDS v3 API

I found adding the following information can help me to understand the relationship between xDS data structure. It is easy to get lost.  Especially for a Envoy newbie. Note that `static_resources` is not used in xDS protocol instead it uses `dynamic_resources`. Yet xDS share the same proto (data structure) with static configuration.  

- LDS: [Listener](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener.proto#config-listener-v3-listener) -> [filter_chains](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filterchain) -> [filters](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/listener/v3/listener_components.proto#envoy-v3-api-msg-config-listener-v3-filter) -> [HTTP connection manager](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-msg-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager) -> [route_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route.proto#envoy-v3-api-msg-config-route-v3-routeconfiguration) RouteConfiguration
- RDS: [RouteConfiguration](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route.proto#envoy-v3-api-msg-config-route-v3-routeconfiguration) -> [virtual_hosts](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-virtualhost) -> [routes](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-route) -> [route](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-msg-config-route-v3-routeaction) -> cluster: name

Example YAML:  

```yaml
static_resources:

  listeners:
  - name: listener_0
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
                route:
                  host_rewrite_literal: www.envoyproxy.io
                  cluster: service_envoyproxy_io
```

- CDS: [Cluster](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#config-cluster-v3-cluster) -> [load_assignment](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint.proto#envoy-v3-api-msg-config-endpoint-v3-clusterloadassignment) ClusterLoadAssignment
- EDS: [ClusterLoadAssignment](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint.proto#envoy-v3-api-msg-config-endpoint-v3-clusterloadassignment) -> [endpoints](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-msg-config-endpoint-v3-localitylbendpoints) -> [lb_endpoints](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-msg-config-endpoint-v3-lbendpoint) -> [endpoint](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/endpoint/v3/endpoint_components.proto#envoy-v3-api-msg-config-endpoint-v3-endpoint) -> [address](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/address.proto#envoy-v3-api-msg-config-core-v3-address) -> [socket_address](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/address.proto#envoy-v3-api-msg-config-core-v3-socketaddress)

Example YAML:  

```yaml
                  cluster: service_envoyproxy_io

  clusters:
  - name: service_envoyproxy_io
    connect_timeout: 30s
    type: LOGICAL_DNS
    # Comment out the following line to test on v6 networks
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: service_envoyproxy_io
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: www.envoyproxy.io
                port_value: 443
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: www.envoyproxy.io
```

## xDS bootstrap file

gRPC uses `XdsClient` to interact with xDS management server. `XdsClient` need a bootstrap file which is a JSON file. The bootstrap file is determined via the `GRPC_XDS_BOOTSTRAP` environment variable.  

The following is the definition of bootstrap JSON file.

```go
type bootstrapConfig struct {                                
    XdsServers               []server                   `json:"xds_servers,omitempty"`                                
    Node                     node                       `json:"node,omitempty"`                                
    CertificateProviders     map[string]json.RawMessage `json:"certificate_providers,omitempty"`                                
    GRPCServerResourceNameID string                     `json:"grpc_server_resource_name_id,omitempty"`                                
}                                                                            
                                 
type server struct {                                
    ServerURI      string   `json:"server_uri,omitempty"`                                            
    ChannelCreds   []creds  `json:"channel_creds,omitempty"`                                
    ServerFeatures []string `json:"server_features,omitempty"`                                
}                                      
                                                                                           
type creds struct {                                                
    Type   string      `json:"type,omitempty"`                                             
    Config interface{} `json:"config,omitempty"`                                
}                                
                                
type node struct {                                                                                    
    ID string `json:"id,omitempty"`                                
}                                                                                    

```

The example of the gRPC xDS bootstrap file.

```json
{
   "xds_server": {
     "server_uri": <string containing URI of management server>,
     "channel_creds": [
       {
         "type": <string containing channel cred type>,
         "config": <JSON object containing config for the type>
       }
     ],
     "server_features": [ ... ],
   },
   "node": <JSON form of Node proto>,
   "certificate_providers" : {
     "default": {
       "plugin_name": "default-plugin-name",
       "config": { default plugin config in JSON }
      },
     "foo": {
       "plugin_name": "foo",
       "config": { foo plugin config in JSON }
     }
   },
   "grpc_server_resource_name_id": "grpc/server"
}


```

Comparing it with the Envoy bootstrap file for ADS.  

```yaml
node:
  cluster: test-cluster
  id: test-id

dynamic_resources:
  ads_config:
    api_type: GRPC
    transport_api_version: V3
    grpc_services:
    - envoy_grpc:
        cluster_name: xds_cluster
  cds_config:
    resource_api_version: V3
    ads: {}
  lds_config:
    resource_api_version: V3
    ads: {}

static_resources:
  clusters:
  - connect_timeout: 1s
    type: strict_dns
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
        explicit_http_config:
          http2_protocol_options: {}
    name: xds_cluster
    load_assignment:
      cluster_name: xds_cluster
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: my-control-plane
                port_value: 18000

admin:
  access_log_path: /dev/null
  address:
    socket_address:
      address: 0.0.0.0
```
