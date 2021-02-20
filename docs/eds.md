# Load Balancing - xDS protocol

- [Initialize EDS balancer](#initialize-eds-balancer)
  - [Send CDS request](#send-cds-request)
  - [Process CDS response](#process-cds-response)
  - [CDS callback](#cds-callback)

In the previous article [Initialize CDS balancer](cds.md#initialize-cds-balancer), we discussed how to initialize CDS balancer and the cluster manager. In this article we will discuss the CDS balancer and CDS request. Load Balancing in xDS is a powerful, flexible and complex tool. There will be more articles to discuss this topic.

## Initialize EDS balancer

In this stage, we continue the discussion of xDS protocol: CDS request and response. Now the CDS balancer got the cluster name. It's the time to Send the CDS request. Here is the map for this stage. In this map:

- Yellow box represents the important type and method/function.
- Green box represents a function run in a dedicated goroutine.
- Arrow represents the call direction and order.
- Grey bar and arrow represents the channel communication for `b.updateCh`.
- Blue bar and arrow represents the channel communication for `t.sendCh`.
- Yellow bar and arrow represents the channel communication for `x.grpcUpdate`.
- Dot line represents the indirect relationship between two boxes.
- Left red dot represents the box is a continue part from other map.
- Right red dot represents there is a extension map for that box.

![xDS protocol: 4](../images/images.013.png)

### Send CDS request

In [Notify CDS balancer](cds.md#notify-cds-balancer), `UpdateClientConnState()` sends the `&ccUpdate{clusterName: lbCfg.ClusterName}` to `b.updateCh`. `cdsBalancer.run()` is waiting for the message from `b.updateCh.Get()`.

Upon receive the `ccUpdate` message, `b.handleClientConnUpdate()` is called to process the new message.

- `b.handleClientConnUpdate()` is actually `cdsBalancer.handleClientConnUpdate()`.
- `handleClientConnUpdate()` calls `b.xdsClient.WatchCluster()` to start watching on resource name `update.clusterName`.
- Please note that `cdsCallback` is `b.handleClusterUpdate`, which is actually `cdsBalancer.handleClusterUpdate()`.
- Please refer to [Communicate with xDS server](xds.md#communicate-with-xds-server) to understand how to start resource watching.
- `WatchCluster()` sends a CDS request to xDS server through `TransportHelper.send()`.
- `TransportHelper.recv()` receives CDS response and calls `handleCDSResponse()` to pre-process the raw CDS response.

Next, we continue the discussion with `handleCDSResponse()`.

```go
// run is a long-running goroutine which handles all updates from gRPC. All
// methods which are invoked directly by gRPC or xdsClient simply push an
// update onto a channel which is read and acted upon right here.
func (b *cdsBalancer) run() {
    for {
        select {
        case u := <-b.updateCh.Get():
            b.updateCh.Load()
            switch update := u.(type) {
            case *ccUpdate:
                b.handleClientConnUpdate(update)
            case *scUpdate:
                // SubConn updates are passthrough and are simply handed over to
                // the underlying edsBalancer.
                if b.edsLB == nil {
                    b.logger.Errorf("xds: received scUpdate {%+v} with no edsBalancer", update)
                    break
                }
                b.edsLB.UpdateSubConnState(update.subConn, update.state)
            case *watchUpdate:
                b.handleWatchUpdate(update)
            }

        // Close results in cancellation of the CDS watch and closing of the
        // underlying edsBalancer and is the only way to exit this goroutine.
        case <-b.closed.Done():
            b.cancelWatch()
            b.cancelWatch = func() {}

            if b.edsLB != nil {
                b.edsLB.Close()
                b.edsLB = nil
            }
            // This is the *ONLY* point of return from this function.
            b.logger.Infof("Shutdown")
            return
        }
    }
}

// handleClientConnUpdate handles a ClientConnUpdate received from gRPC. Good
// updates lead to registration of a CDS watch. Updates with error lead to
// cancellation of existing watch and propagation of the same error to the
// edsBalancer.
func (b *cdsBalancer) handleClientConnUpdate(update *ccUpdate) {
    // We first handle errors, if any, and then proceed with handling the
    // update, only if the status quo has changed.
    if err := update.err; err != nil {
        b.handleErrorFromUpdate(err, true)
    }
    if b.clusterToWatch == update.clusterName {
        return
    }
    if update.clusterName != "" {
        cancelWatch := b.xdsClient.WatchCluster(update.clusterName, b.handleClusterUpdate)
        b.logger.Infof("Watch started on resource name %v with xds-client %p", update.clusterName, b.xdsClient)
        b.cancelWatch = func() {
            cancelWatch()
            b.logger.Infof("Watch cancelled on resource name %v with xds-client %p", update.clusterName, b.xdsClient)
        }
        b.clusterToWatch = update.clusterName
    }
}

// WatchCluster uses CDS to discover information about the provided
// clusterName.
//
// WatchCluster can be called multiple times, with same or different
// clusterNames. Each call will start an independent watcher for the resource.
//
// Note that during race (e.g. an xDS response is received while the user is
// calling cancel()), there's a small window where the callback can be called
// after the watcher is canceled. The caller needs to handle this case.
func (c *clientImpl) WatchCluster(clusterName string, cb func(ClusterUpdate, error)) (cancel func()) {
    wi := &watchInfo{
        c:           c,
        rType:       ClusterResource,
        target:      clusterName,
        cdsCallback: cb,
    }

    wi.expiryTimer = time.AfterFunc(c.watchExpiryTimeout, func() {
        wi.timeout()
    })
    return c.watch(wi)
}

// handleClusterUpdate is the CDS watch API callback. It simply pushes the
// received information on to the update channel for run() to pick it up.
func (b *cdsBalancer) handleClusterUpdate(cu xdsclient.ClusterUpdate, err error) {
    if b.closed.HasFired() {
        b.logger.Warningf("xds: received cluster update {%+v} after cdsBalancer was closed", cu)
        return
    }
    b.updateCh.Put(&watchUpdate{cds: cu, err: err})
}
```

### Process CDS response

`handleCDSResponse()` calls `xdsclient.UnmarshalCluster()` to transform CDS `DiscoveryResponse` into `ClusterUpdate`.

- `UnmarshalCluster()` calls `proto.Unmarshal()` to unmarshal the `DiscoveryResponse` response.
- `UnmarshalCluster()` calls `validateCluster()` to extract supported fields from the `Cluster` object.
  - Please refer to [config.cluster.v3.Cluster](https://github.com/envoyproxy/envoy/blob/b77157803df9a1e6dff53cc616b32ddbf79f83f2/api/envoy/config/cluster/v3/cluster.proto#L43) to understand the meaning of the following fields.
  - `validateCluster()` checks that the `type` field must be [EDS](https://github.com/envoyproxy/envoy/blob/b77157803df9a1e6dff53cc616b32ddbf79f83f2/api/envoy/config/cluster/v3/cluster.proto#L48)
  - `validateCluster()` checks that the `eds_cluster_config` field should be [ADS](https://github.com/envoyproxy/envoy/blob/b77157803df9a1e6dff53cc616b32ddbf79f83f2/api/envoy/config/core/v3/config_source.proto#L151).
  - `validateCluster()` checks that the `lb_policy` field should be [ROUND_ROBIN](https://github.com/envoyproxy/envoy/blob/b77157803df9a1e6dff53cc616b32ddbf79f83f2/api/envoy/config/cluster/v3/cluster.proto#L75).
  - `validateCluster()` calls `securityConfigFromCluster()` to build the `SecurityConfig` object from the [transport_socket](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/core/v3/base.proto#envoy-v3-api-msg-config-core-v3-transportsocket) field.
  - `SecurityConfig` is custom transport socket implementation to use for upstream connections.
  - `validateCluster()` calls `circuitBreakersFromCluster()` to extract the [max_requests](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit_breaker.proto#envoy-v3-api-msg-config-cluster-v3-circuitbreakers-thresholds) from the [circuit_breakers
](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit_breaker.proto#envoy-v3-api-msg-config-cluster-v3-circuitbreakers) field
  - `validateCluster()` checks that the `eds_config` field of [eds_cluster_config](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/cluster.proto#envoy-v3-api-msg-config-cluster-v3-cluster-edsclusterconfig) may has a optional `service_name`. That name is used as the `ServiceName` in `ClusterUpdate`.
  - `validateCluster()` checks that the `lrs_server` field is set. If set, the `EnableLRS` field in `ClusterUpdate` is true.
  - Please note that the `lrs_server` is not implemented at the time of writing this chapter, at least for envoy proxy.
- If the Cluster message in the CDS response did not contain a serviceName, just use the clusterName for EDS.
- `handleCDSResponse()` calls `v3c.parent.NewClusters`, which is actually `clientImpl.NewClusters()` method.
  - For each `ClusterUpdate`, `NewClusters()` checks if it exists in `c.cdsWatchers`, if it does exist, calls `wi.newUpdate()` and add it to `c.cdsCache`.
    - `wi.newUpdate()` updates the `wi.state`, stops the `wi.expiryTimer` and calls `wi.c.scheduleCallback()`.
  - If resource exists in cache, but not in the new update, delete it from cache and call `wi.resourceNotFound()`.
    - `wi.resourceNotFound()` updates the `wi.state`, stops the `wi.expiryTimer.Stop` and calls `wi.sendErrorLocked()`.
    - `wi.sendErrorLocked()` clears the `update` object and calls `wi.c.scheduleCallback()`.
  - `scheduleCallback()` sends `watcherInfoWithUpdate` message to channel `c.updateCh`.

Now the `watcherInfoWithUpdate` has been sent to channel `c.updateCh`. Let's discuss the receiving part in next chapter.

```go
// handleCDSResponse processes an CDS response received from the management
// server. On receipt of a good response, it also invokes the registered watcher
// callback.
func (v3c *client) handleCDSResponse(resp *v3discoverypb.DiscoveryResponse) error {
    update, err := xdsclient.UnmarshalCluster(resp.GetResources(), v3c.logger)
    if err != nil {
        return err
    }
    v3c.parent.NewClusters(update)
    return nil
}

// UnmarshalCluster processes resources received in an CDS response, validates
// them, and transforms them into a native struct which contains only fields we
// are interested in.
func UnmarshalCluster(resources []*anypb.Any, logger *grpclog.PrefixLogger) (map[string]ClusterUpdate, error) {
    update := make(map[string]ClusterUpdate)
    for _, r := range resources {
        if !IsClusterResource(r.GetTypeUrl()) {
            return nil, fmt.Errorf("xds: unexpected resource type: %q in CDS response", r.GetTypeUrl())
        }

        cluster := &v3clusterpb.Cluster{}
        if err := proto.Unmarshal(r.GetValue(), cluster); err != nil {
            return nil, fmt.Errorf("xds: failed to unmarshal resource in CDS response: %v", err)
        }
        logger.Infof("Resource with name: %v, type: %T, contains: %v", cluster.GetName(), cluster, cluster)
        cu, err := validateCluster(cluster)
        if err != nil {
            return nil, err
        }

        // If the Cluster message in the CDS response did not contain a
        // serviceName, we will just use the clusterName for EDS.
        if cu.ServiceName == "" {
            cu.ServiceName = cluster.GetName()
        }
        logger.Debugf("Resource with name %v, value %+v added to cache", cluster.GetName(), cu)
        update[cluster.GetName()] = cu
    }
    return update, nil
}

func validateCluster(cluster *v3clusterpb.Cluster) (ClusterUpdate, error) {
    emptyUpdate := ClusterUpdate{ServiceName: "", EnableLRS: false}
    switch {
    case cluster.GetType() != v3clusterpb.Cluster_EDS:
        return emptyUpdate, fmt.Errorf("xds: unexpected cluster type %v in response: %+v", cluster.GetType(), cluster)
    case cluster.GetEdsClusterConfig().GetEdsConfig().GetAds() == nil:
        return emptyUpdate, fmt.Errorf("xds: unexpected edsConfig in response: %+v", cluster)
    case cluster.GetLbPolicy() != v3clusterpb.Cluster_ROUND_ROBIN:
        return emptyUpdate, fmt.Errorf("xds: unexpected lbPolicy %v in response: %+v", cluster.GetLbPolicy(), cluster)
    }

    sc, err := securityConfigFromCluster(cluster)
    if err != nil {
        return emptyUpdate, err
    }
    return ClusterUpdate{
        ServiceName: cluster.GetEdsClusterConfig().GetServiceName(),
        EnableLRS:   cluster.GetLrsServer().GetSelf() != nil,
        SecurityCfg: sc,
        MaxRequests: circuitBreakersFromCluster(cluster),
    }, nil
}

// securityConfigFromCluster extracts the relevant security configuration from
// the received Cluster resource.
func securityConfigFromCluster(cluster *v3clusterpb.Cluster) (*SecurityConfig, error) {
    // The Cluster resource contains a `transport_socket` field, which contains
    // a oneof `typed_config` field of type `protobuf.Any`. The any proto
    // contains a marshaled representation of an `UpstreamTlsContext` message.
    ts := cluster.GetTransportSocket()
    if ts == nil {
        return nil, nil
    }
    if name := ts.GetName(); name != transportSocketName {
        return nil, fmt.Errorf("xds: transport_socket field has unexpected name: %s", name)
    }
    any := ts.GetTypedConfig()
    if any == nil || any.TypeUrl != version.V3UpstreamTLSContextURL {
        return nil, fmt.Errorf("xds: transport_socket field has unexpected typeURL: %s", any.TypeUrl)
    }
    upstreamCtx := &v3tlspb.UpstreamTlsContext{}
    if err := proto.Unmarshal(any.GetValue(), upstreamCtx); err != nil {
        return nil, fmt.Errorf("xds: failed to unmarshal UpstreamTlsContext in CDS response: %v", err)
    }
    if upstreamCtx.GetCommonTlsContext() == nil {
        return nil, errors.New("xds: UpstreamTlsContext in CDS response does not contain a CommonTlsContext")
    }

    sc, err := securityConfigFromCommonTLSContext(upstreamCtx.GetCommonTlsContext())
    if err != nil {
        return nil, err
    }
    if sc.RootInstanceName == "" {
        return nil, errors.New("security configuration on the client-side does not contain root certificate provider instance name")
    }
    return sc, nil
}

// circuitBreakersFromCluster extracts the circuit breakers configuration from
// the received cluster resource. Returns nil if no CircuitBreakers or no
// Thresholds in CircuitBreakers.
func circuitBreakersFromCluster(cluster *v3clusterpb.Cluster) *uint32 {
    if !env.CircuitBreakingSupport {
        return nil
    }
    for _, threshold := range cluster.GetCircuitBreakers().GetThresholds() {
        if threshold.GetPriority() != v3corepb.RoutingPriority_DEFAULT {
            continue
        }
        maxRequestsPb := threshold.GetMaxRequests()
        if maxRequestsPb == nil {
            return nil
        }
        maxRequests := maxRequestsPb.GetValue()
        return &maxRequests
    }
    return nil
}

// NewClusters is called by the underlying xdsAPIClient when it receives an xDS
// response.
//
// A response can contain multiple resources. They will be parsed and put in a
// map from resource name to the resource content.
func (c *clientImpl) NewClusters(updates map[string]ClusterUpdate) {
    c.mu.Lock()
    defer c.mu.Unlock()

    for name, update := range updates {
        if s, ok := c.cdsWatchers[name]; ok {
            for wi := range s {
                wi.newUpdate(update)
            }
            // Sync cache.
            c.logger.Debugf("CDS resource with name %v, value %+v added to cache", name, update)
            c.cdsCache[name] = update
        }
    }
    for name := range c.cdsCache {
        if _, ok := updates[name]; !ok {
            // If resource exists in cache, but not in the new update, delete it
            // from cache, and also send an resource not found error to indicate
            // resource removed.
            delete(c.cdsCache, name)
            for wi := range c.cdsWatchers[name] {
                wi.resourceNotFound()
            }
        }
    }
    // When CDS resource is removed, we don't delete corresponding EDS cached
    // data. The EDS watch will be canceled, and cache entry is removed when the
    // last watch is canceled.
}
```

### CDS callback

Now we got a message in channel `c.updateCh`, It's time to consume it. From [xDS callback](xds.md#xds-callback), we know the following process will happen:

- `clientImpl.run()` calls `c.callCallback()` with the incoming `watcherInfoWithUpdate` as parameter.
- In our case, the message's `wi.rType` field is `ClusterResource`.
- `callCallback()` will call `cdsCallback()`, which actually calls `cdsBalancer.handleClusterUpdate()`.
- `handleClusterUpdate()` just wraps the `xdsclient.ClusterUpdate` object in `&watchUpdate{cds: cu, err: err}` and sends `watchUpdate` to `b.updateCh`.

```go
// handleClusterUpdate is the CDS watch API callback. It simply pushes the
// received information on to the update channel for run() to pick it up.
func (b *cdsBalancer) handleClusterUpdate(cu xdsclient.ClusterUpdate, err error) {
    if b.closed.HasFired() {
        b.logger.Warningf("xds: received cluster update {%+v} after cdsBalancer was closed", cu)
        return
    }
    b.updateCh.Put(&watchUpdate{cds: cu, err: err})
}
```
