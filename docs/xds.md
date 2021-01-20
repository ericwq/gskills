
# xDS protocol support

- [Protocol buffers map for xDS v3 API](#protocol-buffers-map-for-xds-v3-api)
- [xDS bootstrap file](#xds-bootstrap-file)
- [`xdsResolverBuilder`](#xdsresolverbuilder)
- [Prepare `Config` from bootstrap file](##prepare-config-from-bootstrap-file)
- [Dial to xDS server](#dial-to-xds-server)
- [Process the update from xDS server](#process-the-update-from-xds-server)

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
   "xds_server": [
     {
       "server_uri": <string containing URI of management server>,
       "channel_creds": [
         {
           "type": <string containing channel cred type>,
           "config": <JSON object containing config for the type>
         }
       ],
       "server_features": [ ... ],
     }
   ],
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

Comparing it with the Envoy bootstrap file for ADS.  The `node` field in xDS bootstrap file will be populated with the xDS [Node proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#config-core-v3-node).

Add more information about `certificate_providers` and `grpc_server_resource_name_id` and `xds_server` TODO

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

## `xdsResolverBuilder`

If the user application uses `xds:///example.grpc.io` as target URI, gRPC will start the `XdsClient` initialization process. `xdsResolverBuilder.Build()` will be called to prepare for the interaction with xDS server and return the resolver to gRPC.

- Calls `newXDSClient()` to build the `XdsClient`.
- Calls `watchService()` to register a watch? TODO
- Starts a goroutine `r.run()` to receive the service update. TODO

Let's discuss the `newXDSClient()` in detail next.

```go
const xdsScheme = "xds"

// For overriding in unittests.
var newXDSClient = func() (xdsClientInterface, error) { return xdsclient.New() }

func init() {
    resolver.Register(&xdsResolverBuilder{})
}

type xdsResolverBuilder struct{}                                
                                
// Build helps implement the resolver.Builder interface.                                
//                                
// The xds bootstrap process is performed (and a new xds client is built) every                                
// time an xds resolver is built.                                
func (b *xdsResolverBuilder) Build(t resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {                                
    r := &xdsResolver{                                
        target:         t,                                
        cc:             cc,                                
        closed:         grpcsync.NewEvent(),                                
        updateCh:       make(chan suWithError, 1),                                
        activeClusters: make(map[string]*clusterInfo),                                
    }                                
    r.logger = prefixLogger((r))                                
    r.logger.Infof("Creating resolver for target: %+v", t)                                
                                
    client, err := newXDSClient()                                
    if err != nil {                                
        return nil, fmt.Errorf("xds: failed to create xds-client: %v", err)                                
    }                                
    r.client = client                                
                                
    // If xds credentials were specified by the user, but bootstrap configs do                                
    // not contain any certificate provider configuration, it is better to fail                                
    // right now rather than failing when attempting to create certificate                                
    // providers after receiving an CDS response with security configuration.                                
    var creds credentials.TransportCredentials                                
    switch {                                
    case opts.DialCreds != nil:                                
        creds = opts.DialCreds                                
    case opts.CredsBundle != nil:                                
        creds = opts.CredsBundle.TransportCredentials()                                
    }                                
    if xc, ok := creds.(interface{ UsesXDS() bool }); ok && xc.UsesXDS() {                                
        bc := client.BootstrapConfig()                                
        if len(bc.CertProviderConfigs) == 0 {
            return nil, errors.New("xds: xdsCreds specified but certificate_providers config missing in bootstrap file")
        }
    }

    // Register a watch on the xdsClient for the user's dial target.
    cancelWatch := watchService(r.client, r.target.Endpoint, r.handleServiceUpdate, r.logger)
    r.logger.Infof("Watch started on resource name %v with xds-client %p", r.target.Endpoint, r.client)
    r.cancelWatch = func() {
        cancelWatch()
        r.logger.Infof("Watch cancel on resource name %v with xds-client %p", r.target.Endpoint, r.client)
    }

    go r.run()
    return r, nil
}

// Name helps implement the resolver.Builder interface.
func (*xdsResolverBuilder) Scheme() string {
    return xdsScheme
}

```

## Prepare `Config` from bootstrap file

`newXDSClient()` is a wrapper for `Client.New()`. `Client` is a singleton and a wrapper for `clientImpl` and maintains the `refcount`.

- `Client.New()` calls `bootstrapNewConfig()` to read the file specified in `GRPC_XDS_BOOTSTRAP` environment variable. It returns a `bootstrap.Config`. Which means `bootstrapNewConfig()` reads the bootstrap file and converts the content into `bootstrap.Config`. `bootstrap.Config` is an internal data structure for xDS client.
  - `bootstrapNewConfig()` is actually `bootstrap.NewConfig()`, which in turn is `Config.NewConfig()`.
  - `Config.NewConfig()` calls `bootstrapConfigFromEnvVariable()` to read the file specified in either `GRPC_XDS_BOOTSTRAP` or `GRPC_XDS_BOOTSTRAP_CONFIG`
  - Here we does not show the code of `bootstrapConfigFromEnvVariable()`. It's simple enough.
  - `Config.NewConfig()` use JSON decoder to decode the bootstrap data into:
    - `config.NodeProto` copies from `node`, note: the `node` data is transformed from JSON to `v3.Node proto`.
    - `config.BalancerName` copies from the first `server_uri`,
    - `config.Creds` comes from the first `channel_creds`,
    - `config.CertProviderConfigs` comes from `certificate_providers`
    - `config.ServerResourceNameID` comes from `grpc_server_resource_name_id`
  - `config.TransportAPI` is `version.TransportV2` by default. If the Server supports v3 and `GRPC_XDS_EXPERIMENTAL_V3_SUPPORT` is true, `config.TransportAPI` is `version.TransportV3`
  - `Config.NewConfig()` calls `config.updateNodeProto()` to automatically fill in some fields of `NodeProto`.
    - `config.updateNodeProto()` supports both xDS v2 and v3. We will explain it in v3 because v2 is no longer support by Envoy.
    - `UserAgentName` get the value `"gRPC Go"`
    - `UserAgentVersionType` get the value `{"userAgentVersion":"1.36.0-dev"}`
    - `ClientFeatures` get the value `"envoy.lb.does_not_support_overprovisioning"`
- After `bootstrap.Config` is ready, `Client.New()` calls `newWithConfig()` to use the `bootstrap.Config` to create a `clientImpl`, which is the real `XdsClient`.
- It's long enough. Let's discuss `newWithConfig()` in next part.

```go
// This is the Client returned by New(). It contains one client implementation,
// and maintains the refcount.
var singletonClient = &Client{}

// To override in tests.
var bootstrapNewConfig = bootstrap.NewConfig

// Client is a full fledged gRPC client which queries a set of discovery APIs
// (collectively termed as xDS) on a remote management server, to discover
// various dynamic resources.
//
// The xds client is a singleton. It will be shared by the xds resolver and
// balancer implementations, across multiple ClientConns and Servers.
type Client struct {                 
    *clientImpl                                           

    // This mu protects all the fields, including the embedded clientImpl above.
    mu       sync.Mutex                                   
    refCount int
}
                       
// New returns a new xdsClient configured by the bootstrap file specified in env
// variable GRPC_XDS_BOOTSTRAP.
func [`New`](#New)() (*Client, error) {                                                   
    singletonClient.mu.Lock()
    defer singletonClient.mu.Unlock()
    // If the client implementation was created, increment ref count and return
    // the client.
    if singletonClient.clientImpl != nil {
        singletonClient.refCount++
        return singletonClient, nil
    }                                                   
  
    // Create the new client implementation.
    config, err := bootstrapNewConfig()
    if err != nil {
        return nil, fmt.Errorf("xds: failed to read bootstrap file: %v", err)
    }
    c, err := newWithConfig(config, defaultWatchExpiryTimeout)
    if err != nil {                                                                                                               
        return nil, err                                                                                                           
    }                                                                                                                             
                                                                                                                                  
    singletonClient.clientImpl = c                                                                                                
    singletonClient.refCount++                                                                                                    
    return singletonClient, nil                                                                                                   
}                                                                                                                                 

// NewConfig returns a new instance of Config initialized by reading the
// bootstrap file found at ${GRPC_XDS_BOOTSTRAP}.
//
// The format of the bootstrap file will be as follows:
// {
//    "xds_server": {
//      "server_uri": <string containing URI of management server>,
//      "channel_creds": [
//        {
//          "type": <string containing channel cred type>,
//          "config": <JSON object containing config for the type>
//        }
//      ],
//      "server_features": [ ... ],
//    },
//    "node": <JSON form of Node proto>,
//    "certificate_providers" : {
//      "default": {
//        "plugin_name": "default-plugin-name",
//        "config": { default plugin config in JSON }
//       },
//      "foo": {
//        "plugin_name": "foo",
//        "config": { foo plugin config in JSON }
//      }
//    },
//    "grpc_server_resource_name_id": "grpc/server"
// }
//
// Currently, we support exactly one type of credential, which is
// "google_default", where we use the host's default certs for transport
// credentials and a Google oauth token for call credentials.
//
// This function tries to process as much of the bootstrap file as possible (in
// the presence of the errors) and may return a Config object with certain
// fields left unspecified, in which case the caller should use some sane
// defaults.
func NewConfig() (*Config, error) {
    config := &Config{}

    data, err := bootstrapConfigFromEnvVariable()
    if err != nil {
        return nil, fmt.Errorf("xds: Failed to read bootstrap config: %v", err)
    }
    logger.Debugf("Bootstrap content: %s", data)

    var jsonData map[string]json.RawMessage
    if err := json.Unmarshal(data, &jsonData); err != nil {
        return nil, fmt.Errorf("xds: Failed to parse bootstrap config: %v", err)
    }

    serverSupportsV3 := false
    m := jsonpb.Unmarshaler{AllowUnknownFields: true}
    for k, v := range jsonData {
        switch k {
        case "node":
            // We unconditionally convert the JSON into a v3.Node proto. The v3
            // proto does not contain the deprecated field "build_version" from
            // the v2 proto. We do not expect the bootstrap file to contain the
            // "build_version" field. In any case, the unmarshal will succeed
            // because we have set the `AllowUnknownFields` option on the
            // unmarshaler.
            n := &v3corepb.Node{}
            if err := m.Unmarshal(bytes.NewReader(v), n); err != nil {
                return nil, fmt.Errorf("xds: jsonpb.Unmarshal(%v) for field %q failed during bootstrap: %v", string(v), k, err)
            }
            config.NodeProto = n
        case "xds_servers":
            var servers []*xdsServer
            if err := json.Unmarshal(v, &servers); err != nil {
                return nil, fmt.Errorf("xds: json.Unmarshal(%v) for field %q failed during bootstrap: %v", string(v), k, err)
            }
            if len(servers) < 1 {
                return nil, fmt.Errorf("xds: bootstrap file parsing failed during bootstrap: file doesn't contain any management server to       connect to")
            }
            xs := servers[0]
            config.BalancerName = xs.ServerURI
            for _, cc := range xs.ChannelCreds {
                // We stop at the first credential type that we support.
                if cc.Type == credsGoogleDefault {
                    config.Creds = grpc.WithCredentialsBundle(google.NewDefaultCredentials())
                    break
                } else if cc.Type == credsInsecure {
                    config.Creds = grpc.WithTransportCredentials(insecure.NewCredentials())
                    break
                }
            }
            for _, f := range xs.ServerFeatures {
                switch f {
                case serverFeaturesV3:
                    serverSupportsV3 = true
                }
            }
        case "certificate_providers":
            var providerInstances map[string]json.RawMessage
            if err := json.Unmarshal(v, &providerInstances); err != nil {
                return nil, fmt.Errorf("xds: json.Unmarshal(%v) for field %q failed during bootstrap: %v", string(v), k, err)
            }
            configs := make(map[string]*certprovider.BuildableConfig)
            getBuilder := internal.GetCertificateProviderBuilder.(func(string) certprovider.Builder)
            for instance, data := range providerInstances {
                var nameAndConfig struct {
                    PluginName string          `json:"plugin_name"`
                    Config     json.RawMessage `json:"config"`
                }
                if err := json.Unmarshal(data, &nameAndConfig); err != nil {
                    return nil, fmt.Errorf("xds: json.Unmarshal(%v) for field %q failed during bootstrap: %v", string(v), instance, err)
                }

                name := nameAndConfig.PluginName
                parser := getBuilder(nameAndConfig.PluginName)
                if parser == nil {
                    // We ignore plugins that we do not know about.
                    continue
                }
                bc, err := parser.ParseConfig(nameAndConfig.Config)
                if err != nil {
                    return nil, fmt.Errorf("xds: Config parsing for plugin %q failed: %v", name, err)
                }
                configs[instance] = bc
            }
            config.CertProviderConfigs = configs
        case "grpc_server_resource_name_id":
            if err := json.Unmarshal(v, &config.ServerResourceNameID); err != nil {
                return nil, fmt.Errorf("xds: json.Unmarshal(%v) for field %q failed during bootstrap: %v", string(v), k, err)
            }
        }
        // Do not fail the xDS bootstrap when an unknown field is seen. This can
        // happen when an older version client reads a newer version bootstrap
        // file with new fields.
    }

    if config.BalancerName == "" {
        return nil, fmt.Errorf("xds: Required field %q not found in bootstrap %s", "xds_servers.server_uri", jsonData["xds_servers"])
    }
    if config.Creds == nil {
        return nil, fmt.Errorf("xds: Required field %q doesn't contain valid value in bootstrap %s", "xds_servers.channel_creds", jsonData      ["xds_servers"])
    }

    // We end up using v3 transport protocol version only if the following
    // conditions are met:
    // 1. Server supports v3, indicated by the presence of "xds_v3" in
    //    server_features.
    // 2. Environment variable "GRPC_XDS_EXPERIMENTAL_V3_SUPPORT" is set to
    //    true.
    // The default value of the enum type "version.TransportAPI" is v2.
}

// updateNodeProto updates the node proto read from the bootstrap file.
//
// Node proto in Config contains a v3.Node protobuf message corresponding to the
// JSON contents found in the bootstrap file. This method performs some post
// processing on it:
// 1. If we don't find a nodeProto in the bootstrap file, we create an empty one
// here. That way, callers of this function can always expect that the NodeProto
// field is non-nil.
// 2. If the transport protocol version to be used is not v3, we convert the
// current v3.Node proto in a v2.Node proto.
// 3. Some additional fields which are not expected to be set in the bootstrap
// file are populated here.
func (c *Config) updateNodeProto() error {
    if c.TransportAPI == version.TransportV3 {
        v3, _ := c.NodeProto.(*v3corepb.Node)
        if v3 == nil {
            v3 = &v3corepb.Node{}
        }
        v3.UserAgentName = gRPCUserAgentName
        v3.UserAgentVersionType = &v3corepb.Node_UserAgentVersion{UserAgentVersion: grpc.Version}
        v3.ClientFeatures = append(v3.ClientFeatures, clientFeatureNoOverprovisioning)
        c.NodeProto = v3
        return nil
    }

    v2 := &v2corepb.Node{}
    if c.NodeProto != nil {
        v3, err := proto.Marshal(c.NodeProto)
        if err != nil {
            return fmt.Errorf("xds: proto.Marshal(%v): %v", c.NodeProto, err)
        }
        if err := proto.Unmarshal(v3, v2); err != nil {
            return fmt.Errorf("xds: proto.Unmarshal(%v): %v", v3, err)
        }
    }
    c.NodeProto = v2

    // BuildVersion is deprecated, and is replaced by user_agent_name and
    // user_agent_version. But the management servers are still using the old
    // field, so we will keep both set.
    v2.BuildVersion = gRPCVersion
    v2.UserAgentName = gRPCUserAgentName
    v2.UserAgentVersionType = &v2corepb.Node_UserAgentVersion{UserAgentVersion: grpc.Version}
    v2.ClientFeatures = append(v2.ClientFeatures, clientFeatureNoOverprovisioning)
    return nil
}
```

## Dial to xDS server

The goal of `newWithConfig()` is to create the `XdsClient`, which is a gRPC client in nature.

- `config.Creds` is used as `grpc.DialOption` to create the TLS connection with the xDS server.
- `config.BalancerName` is used as target parameter for `grpc.Dial()`, `config.BalancerName` comes from `server_uri`. So `server_uri` is the target xDS server name.
- After `grpc.Dial()` successfully return the `grpc.*ClientConn`, the connection to xDS server is ready.
- Then `newAPIClient()` is called to build the v2 or v3 `APIClient` according to the `config.TransportAPI` parameter.
  - `APIClientBuilder` creates an xDS client for a specific xDS transport protocol. There are two `APIClientBuilder` available: v2 builder and v3 builder.
  - `getAPIClientBuilder()` uses the `config.TransportAPI` as parameter to determine which `APIClientBuilder` to use.
  - We will discuss the v3 `APIClientBuilder` in next chapter. TODO
- After `APIClient` is ready, `c.run()` will be called to start a new goroutine to process the update configuration from xDS server.
  - `c.run()` waits on channel `c.updateCh.Get()` to read the `watcherInfoWithUpdate` message.
  - if a `watcherInfoWithUpdate` is read, `c.callCallback()` will be called.
  - We will discuss the `c.callCallback()` later. TODO

```go
// newWithConfig returns a new xdsClient with the given config.
func newWithConfig(config *bootstrap.Config, watchExpiryTimeout time.Duration) (*clientImpl, error) {
    switch {
    case config.BalancerName == "":
        return nil, errors.New("xds: no xds_server name provided in options")
    case config.Creds == nil:
        return nil, errors.New("xds: no credentials provided in options")
    case config.NodeProto == nil:
        return nil, errors.New("xds: no node_proto provided in options")
    }

    switch config.TransportAPI {
    case version.TransportV2:
        if _, ok := config.NodeProto.(*v2corepb.Node); !ok {
            return nil, fmt.Errorf("xds: Node proto type (%T) does not match API version: %v", config.NodeProto, config.TransportAPI)
        }
    case version.TransportV3:
        if _, ok := config.NodeProto.(*v3corepb.Node); !ok {
            return nil, fmt.Errorf("xds: Node proto type (%T) does not match API version: %v", config.NodeProto, config.TransportAPI)
        }
    }

    dopts := []grpc.DialOption{
        config.Creds,
        grpc.WithKeepaliveParams(keepalive.ClientParameters{
            Time:    5 * time.Minute,
            Timeout: 20 * time.Second,
        }),
    }

    c := &clientImpl{
        done:               grpcsync.NewEvent(),
        config:             config,
        watchExpiryTimeout: watchExpiryTimeout,

        updateCh:    buffer.NewUnbounded(),
        ldsWatchers: make(map[string]map[*watchInfo]bool),
        ldsCache:    make(map[string]ListenerUpdate),
        rdsWatchers: make(map[string]map[*watchInfo]bool),
        rdsCache:    make(map[string]RouteConfigUpdate),
        cdsWatchers: make(map[string]map[*watchInfo]bool),
        cdsCache:    make(map[string]ClusterUpdate),
        edsWatchers: make(map[string]map[*watchInfo]bool),
        edsCache:    make(map[string]EndpointsUpdate),
        lrsClients:  make(map[string]*lrsClient),
    }

    cc, err := grpc.Dial(config.BalancerName, dopts...)
    if err != nil {
        // An error from a non-blocking dial indicates something serious.
        return nil, fmt.Errorf("xds: failed to dial balancer {%s}: %v", config.BalancerName, err)
    }
    c.cc = cc
    c.logger = prefixLogger((c))
    c.logger.Infof("Created ClientConn to xDS management server: %s", config.BalancerName)

    apiClient, err := newAPIClient(config.TransportAPI, cc, BuildOptions{
        Parent:    c,
        NodeProto: config.NodeProto,
        Backoff:   backoff.DefaultExponential.Backoff,
        Logger:    c.logger,
    })
    if err != nil {
        return nil, err
    }
    c.apiClient = apiClient
    c.logger.Infof("Created")
    go c.run()
    return c, nil
}

// Function to be overridden in tests.                                                                                     
var newAPIClient = func(apiVersion version.TransportAPI, cc *grpc.ClientConn, opts BuildOptions) (APIClient, error) {
    cb := getAPIClientBuilder(apiVersion)                                                                   
    if cb == nil {                                                                                                                         
        return nil, fmt.Errorf("no client builder for xDS API version: %v", apiVersion)                                 
    }                                                                                                                                            
    return cb.Build(cc, opts)                                                                                                                 
}                                                                                                                                        

var (                                                                                                                                     
    m = make(map[version.TransportAPI]APIClientBuilder)                                                                  
)                                                                                

// RegisterAPIClientBuilder registers a client builder for xDS transport protocol                                                             
// version specified by b.Version().                                                                                                              
//                                                                                                                                           
// NOTE: this function must only be called during initialization time (i.e. in                                                 
// an init() function), and is not thread-safe. If multiple builders are                                   
// registered for the same version, the one registered last will take effect.                              
func RegisterAPIClientBuilder(b APIClientBuilder) {                                                                                         
    m[b.Version()] = b                                                                                                                            
}                                                                                                                                             
                                                                                                                       
// getAPIClientBuilder returns the client builder registered for the provided                                          
// xDS transport API version.                                                                                              
func getAPIClientBuilder(version version.TransportAPI) APIClientBuilder {                                            
    if b, ok := m[version]; ok {                                                                            
        return b                                                                                                                           
    }                                                                                                                   
    return nil                                                                                                                                   
}                                                                                                                                             

// APIClientBuilder creates an xDS client for a specific xDS transport protocol                                                             
// version.                                                                                                                                       
type APIClientBuilder interface {                                                                                                             
    // Build builds a transport protocol specific implementation of the xDS                                            
    // client based on the provided clientConn to the management server and the                                        
    // provided options.                                                                                                   
    Build(*grpc.ClientConn, BuildOptions) (APIClient, error)                                                         
    // Version returns the xDS transport protocol version used by clients build                             
    // using this builder.                                                                                                                 
    Version() version.TransportAPI                                                                                      
}                                                                                                                                                

// run is a goroutine for all the callbacks.
//
// Callback can be called in watch(), if an item is found in cache. Without this
// goroutine, the callback will be called inline, which might cause a deadlock
// in user's code. Callbacks also cannot be simple `go callback()` because the
// order matters.
func (c *clientImpl) run() {
    for {
        select {
        case t := <-c.updateCh.Get():
            c.updateCh.Load()
            if c.done.HasFired() {
                return
            }
            c.callCallback(t.(*watcherInfoWithUpdate))
        case <-c.done.Done():
            return
        }
    }
}
```

## `newClient()`

Here is the v3 `APIClientBuilder`, its `Build()` method will call `newClient()` to do the job.

- `newClient()` creates a `v3c` object and uses `v3c` as parameter to call `xdsclient.NewTransportHelper()` to get the `TransportHelper`.
- `NewTransportHelper()` creates `TransportHelper` object and start a new goroutine `t.run()`.
  - `TransportHelper.run()` starts an ADS stream. Runs the sender and receiver goroutine to send and receive data from the stream respectively.
  - `TransportHelper.run()` start a new goroutine `t.send()`, it's the sender goroutine to send data to the stream.
  - The remaining part of `TransportHelper.run()` is the receiver goroutine to receive data from the stream.

```go
func init() {                                          
    xdsclient.RegisterAPIClientBuilder(clientBuilder{})
}

var (                                                                                                                        
    resourceTypeToURL = map[xdsclient.ResourceType]string{   
        xdsclient.ListenerResource:    version.V3ListenerURL,                                              
        xdsclient.RouteConfigResource: version.V3RouteConfigURL,                                            
        xdsclient.ClusterResource:     version.V3ClusterURL,                                                                 
        xdsclient.EndpointsResource:   version.V3EndpointsURL,                                                          
    }                                                                                                                                         
)                                                                                                                 

type clientBuilder struct{}                                                                                                          
                                                                                                                                              
func (clientBuilder) Build(cc *grpc.ClientConn, opts xdsclient.BuildOptions) (xdsclient.APIClient, error) {                               
    return newClient(cc, opts)                                                                                                                  
}                                                                                                                         
                                                                                                           
func (clientBuilder) Version() version.TransportAPI {                                                                    
    return version.TransportV3                                                                             
}                                                                                                          
                                                                                                                                                   
func newClient(cc *grpc.ClientConn, opts xdsclient.BuildOptions) (xdsclient.APIClient, error) {            
    nodeProto, ok := opts.NodeProto.(*v3corepb.Node)                                                       
    if !ok {                                                                                               
        return nil, fmt.Errorf("xds: unsupported Node proto type: %T, want %T", opts.NodeProto, v3corepb.Node{})
    }
    v3c := &client{                                                                                         
        cc:        cc,                                                                                                              
        parent:    opts.Parent,                                                                                          
        nodeProto: nodeProto,
        logger:    opts.Logger,                                                                                                                 
    }                                                                                                       
    v3c.ctx, v3c.cancelCtx = context.WithCancel(context.Background())                                                  
    v3c.TransportHelper = xdsclient.NewTransportHelper(v3c, opts.Logger, opts.Backoff)                                          
    return v3c, nil
}

// NewTransportHelper creates a new transport helper to be used by versioned
// client implementations.
func NewTransportHelper(vc VersionedClient, logger *grpclog.PrefixLogger, backoff func(int) time.Duration) *TransportHelper {
    ctx, cancelCtx := context.WithCancel(context.Background())
    t := &TransportHelper{
        cancelCtx: cancelCtx,
        vClient:   vc,
        logger:    logger,
        backoff:   backoff,

        streamCh:   make(chan grpc.ClientStream, 1),
        sendCh:     buffer.NewUnbounded(),
        watchMap:   make(map[ResourceType]map[string]bool),
        versionMap: make(map[ResourceType]string),
        nonceMap:   make(map[ResourceType]string),
    }

    go t.run(ctx)
    return t
}

// run starts an ADS stream (and backs off exponentially, if the previous
// stream failed without receiving a single reply) and runs the sender and
// receiver routines to send and receive data from the stream respectively.
func (t *TransportHelper) run(ctx context.Context) {
    go t.send(ctx)
    // TODO: start a goroutine monitoring ClientConn's connectivity state, and
    // report error (and log) when stats is transient failure.

    retries := 0
    for {
        select {  
        case <-ctx.Done():
            return
        default:
        }

        if retries != 0 {                             
            timer := time.NewTimer(t.backoff(retries))
            select {          
            case <-timer.C:       
            case <-ctx.Done():                        
                if !timer.Stop() {
                    <-timer.C
                }             
                return
            }                     
        }
                                                                     
        retries++                                                            
        stream, err := t.vClient.NewStream(ctx)
        if err != nil {
            t.logger.Warningf("xds: ADS stream creation failed: %v", err)
            continue
        }
        t.logger.Infof("ADS stream created")

        select {
        case <-t.streamCh:
        default:
        }
        t.streamCh <- stream
        if t.recv(stream) {
            retries = 0
        }
    }
}

// send is a separate goroutine for sending watch requests on the xds stream.
//
// It watches the stream channel for new streams, and the request channel for
// new requests to send on the stream.
//
// For each new request (watchAction), it's
//  - processed and added to the watch map
//    - so resend will pick them up when there are new streams
//  - sent on the current stream if there's one
//    - the current stream is cleared when any send on it fails
//
// For each new stream, all the existing requests will be resent.
//
// Note that this goroutine doesn't do anything to the old stream when there's a
// new one. In fact, there should be only one stream in progress, and new one
// should only be created when the old one fails (recv returns an error).
func (t *TransportHelper) send(ctx context.Context) {
    var stream grpc.ClientStream
    for {
        select {                   
        case <-ctx.Done():              
            return                                       
        case stream = <-t.streamCh:
            if !t.sendExisting(stream) {
                // send failed, clear the current stream.
                stream = nil
            }
        case u := <-t.sendCh.Get():
            t.sendCh.Load()                    
                                                   
            var (                            
                target                 []string
                rType                  ResourceType
                version, nonce, errMsg string
                send                   bool
            )                                                             
            switch update := u.(type) {
            case *watchAction:                                                        
                target, rType, version, nonce = t.processWatchInfo(update)
            case *ackAction:               
                target, rType, version, nonce, send = t.processAckInfo(update, stream)
                if !send {             
                    continue                                                          
                }                                                         
                errMsg = update.errMsg                                                
            }                                                                         
            if stream == nil {
                // There's no stream yet. Skip the request. This request
                // will be resent to the new streams. If no stream is
                // created, the watcher will timeout (same as server not
                // sending response back).
                continue                                                
            }                                                           
            if err := t.vClient.SendRequest(stream, target, rType, version, nonce, errMsg); err != nil {
                t.logger.Warningf("ADS request for {target: %q, type: %v, version: %q, nonce: %q} failed: %v", target, rType, version, nonce, err)
                // send failed, clear the current stream.
                stream = nil
            }                                                                                           
        }                                                                                                                                         
    }                                                                                                                                             
}                                                        
 
```

## Process the update from xDS server

```go
func (c *clientImpl) callCallback(wiu *watcherInfoWithUpdate) {
    c.mu.Lock()
    // Use a closure to capture the callback and type assertion, to save one
    // more switch case.
    //
    // The callback must be called without c.mu. Otherwise if the callback calls
    // another watch() inline, it will cause a deadlock. This leaves a small
    // window that a watcher's callback could be called after the watcher is
    // canceled, and the user needs to take care of it.                              
    var ccb func()                        
    switch wiu.wi.rType {
    case ListenerResource:                                   
        if s, ok := c.ldsWatchers[wiu.wi.target]; ok && s[wiu.wi] {
            ccb = func() { wiu.wi.ldsCallback(wiu.update.(ListenerUpdate), wiu.err) }
        }                   
    case RouteConfigResource:
        if s, ok := c.rdsWatchers[wiu.wi.target]; ok && s[wiu.wi] {            
            ccb = func() { wiu.wi.rdsCallback(wiu.update.(RouteConfigUpdate), wiu.err) }
        }                                                                   
    case ClusterResource:
        if s, ok := c.cdsWatchers[wiu.wi.target]; ok && s[wiu.wi] {             
            ccb = func() { wiu.wi.cdsCallback(wiu.update.(ClusterUpdate), wiu.err) }
        }
    case EndpointsResource:
        if s, ok := c.edsWatchers[wiu.wi.target]; ok && s[wiu.wi] {
            ccb = func() { wiu.wi.edsCallback(wiu.update.(EndpointsUpdate), wiu.err) }
        }
    }
    c.mu.Unlock()

    if ccb != nil {
        ccb()
    }
}
```
