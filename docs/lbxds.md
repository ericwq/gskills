
# Load Balancing - xDS

In the previous article [xDS protocol support](xds.md), we discussed the xDS resolver and LDS/RDS. In this article we will explain how the xDS balancer works and CDS/RDS. It's not clear: why to separate LDS and CDS into different stage? Maybe we can answer the question in the end of this article.

According to [Build `ServiceConfig`](xds.md#build-serviceconfig), `r.cc.UpdateState()` will be called in `xdsResolver.sendNewServiceConfig()` once RDS response is processed. `r.cc.UpdateState()` is actually `ccResolverWrapper.UpdateState()`. The parameter of `r.cc.UpdateState()` is `resolver.State`, which is a struct, resolver uses it to notifiy the gRPC core. For xDS, `resolver.State` contains the following fields:

- The `Addresses` field: NOT used by xDS resolver.
- The `ServiceConfig` field: `xdsResolver.sendNewServiceConfig()` builds `ServiceConfig` based on `xdsResolver.activeClusters`.
- The `Attributes` field: `xdsResolver.sendNewServiceConfig()` sets a key/value pair in `Attributes`field.
  - The key is `"grpc.internal.resolver.configSelector"`.
  - The value is `configSelector` which is built by `xdsResolver.newConfigSelector()`.
  - Please refer to [Build `ServiceConfig`](xds.md#build-serviceconfig) for detail.

```go
// State contains the current Resolver state relevant to the ClientConn.
type State struct {
    // Addresses is the latest set of resolved addresses for the target.
    Addresses []Address

    // ServiceConfig contains the result from parsing the latest service
    // config.  If it is nil, it indicates no service config is present or the
    // resolver does not provide service configs.
    ServiceConfig *serviceconfig.ParseResult

    // Attributes contains arbitrary data about the resolver intended for
    // consumption by the load balancing policy.
    Attributes *attributes.Attributes
}

// sendNewServiceConfig prunes active clusters, generates a new service config
// based on the current set of active clusters, and sends an update to the
// channel with that service config and the provided config selector.  Returns
// false if an error occurs while generating the service config and the update
// cannot be sent.
func (r *xdsResolver) sendNewServiceConfig(cs *configSelector) bool {
    // Delete entries from r.activeClusters with zero references;
    // otherwise serviceConfigJSON will generate a config including
    // them.
    r.pruneActiveClusters()

    if cs == nil && len(r.activeClusters) == 0 {
        // There are no clusters and we are sending a failing configSelector.
        // Send an empty config, which picks pick-first, with no address, and
        // puts the ClientConn into transient failure.
        r.cc.UpdateState(resolver.State{ServiceConfig: r.cc.ParseServiceConfig("{}")})
        return true
    }

    // Produce the service config.
    sc, err := serviceConfigJSON(r.activeClusters)
    if err != nil {
        // JSON marshal error; should never happen.
        r.logger.Errorf("%v", err)
        r.cc.ReportError(err)
        return false
    }
    r.logger.Infof("Received update on resource %v from xds-client %p, generated service config: %v", r.target.Endpoint, r.client, sc)

    // Send the update to the ClientConn.
    state := iresolver.SetConfigSelector(resolver.State{
        ServiceConfig: r.cc.ParseServiceConfig(sc),
    }, cs)
    r.cc.UpdateState(state)
    return true
}

const csKey = csKeyType("grpc.internal.resolver.configSelector")

// SetConfigSelector sets the config selector in state and returns the new
// state.
func SetConfigSelector(state resolver.State, cs ConfigSelector) resolver.State {
    state.Attributes = state.Attributes.WithValues(csKey, cs)
    return state
}

```

## UpdateState

In [Dial process part I](dial.md#dial-process-part-i), resolver calls `UpdateState()` to notify gRPC core the resolver state.

- `UpdateState()` calls `ClientConn.updateResolverState()` with parameter `resolver.State`.
- In `updateResolverState()`, in this case, the `s.ServiceConfig.Config` field of `resolver.State` is not nil,
  - calls `iresolver.GetConfigSelector()` to get the `ConfigSelector` from `State.Attributes`.
  - calls `cc.applyServiceConfigAndBalancer()`

```go
func (ccr *ccResolverWrapper) UpdateState(s resolver.State) {
    if ccr.done.HasFired() {
        return
    }
    channelz.Infof(logger, ccr.cc.channelzID, "ccResolverWrapper: sending update to cc: %v", s)
    if channelz.IsOn() {
        ccr.addChannelzTraceEvent(s)
    }
    ccr.curState = s
    ccr.poll(ccr.cc.updateResolverState(ccr.curState, nil))
}

```

```go
    r := &xdsResolver{activeClusters: map[string]*clusterInfo{
        "zero": {refCount: 0},
        "one":  {refCount: 1},
        "two":  {refCount: 2},
    }}

    // to service config
    result, err := serviceConfigJSON(r.activeClusters)
```

```json
{
  "loadBalancingConfig": [
    {
      "xds_cluster_manager_experimental": {
        "children": {
          "one": {
            "childPolicy": [
              {
                "cds_experimental": {
                  "cluster": "one"
                }
              }
            ]
          },
          "two": {
            "childPolicy": [
              {
                "cds_experimental": {
                  "cluster": "two"
                }
              }
            ]
          },
          "zero": {
            "childPolicy": [
              {
                "cds_experimental": {
                  "cluster": "zero"
                }
              }
            ]
          }
        }
      }
    }
  ]
}
```
