# LoadBalancingConfig JSON parsing

The `LoadBalancingConfig` field in `ServiceConfig` is complex. This is a separated chapter to discuss the implementation of `LoadBalancingConfig` JSON parsing.

The following is the definition of `ServiceConfig`. Note the `lbConfig` is actually the `LoadBalancingConfig`, you will see more detail later. In this chapter we will focus on the `lbConfig` field. Other fields of `ServiceConfig` will be ignored.

```go
type lbConfig struct {
    name string
    cfg  serviceconfig.LoadBalancingConfig
}

// MethodConfig defines the configuration recommended by the service providers for a
// particular method.
//
// Deprecated: Users should not use this struct. Service config should be received
// through name resolver, as specified here
// https://github.com/grpc/grpc/blob/master/doc/service_config.md
type MethodConfig = internalserviceconfig.MethodConfig

// ServiceConfig is provided by the service provider and contains parameters for how
// clients that connect to the service should behave.
//
// Deprecated: Users should not use this struct. Service config should be received
// through name resolver, as specified here
// https://github.com/grpc/grpc/blob/master/doc/service_config.md
type ServiceConfig struct {
    serviceconfig.Config

    // LB is the load balancer the service providers recommends. The balancer
    // specified via grpc.WithBalancerName will override this.  This is deprecated;
    // lbConfigs is preferred.  If lbConfig and LB are both present, lbConfig
    // will be used.
    LB *string

    // lbConfig is the service config's load balancing configuration.  If
    // lbConfig and LB are both present, lbConfig will be used.
    lbConfig *lbConfig

    // Methods contains a map for the methods in this service.  If there is an
    // exact match for a method (i.e. /service/method) in the map, use the
    // corresponding MethodConfig.  If there's no exact match, look for the
    // default config for the service (/service/) and use the corresponding
    // MethodConfig if it exists.  Otherwise, the method has no MethodConfig to
    // use.
    Methods map[string]MethodConfig

    // If a retryThrottlingPolicy is provided, gRPC will automatically throttle
    // retry attempts and hedged RPCs when the client’s ratio of failures to
    // successes exceeds a threshold.
    //
    // For each server name, the gRPC client will maintain a token_count which is
    // initially set to maxTokens, and can take values between 0 and maxTokens.
    //
    // Every outgoing RPC (regardless of service or method invoked) will change
    // token_count as follows:
    //
    //   - Every failed RPC will decrement the token_count by 1.
    //   - Every successful RPC will increment the token_count by tokenRatio.
    //
    // If token_count is less than or equal to maxTokens / 2, then RPCs will not
    // be retried and hedged RPCs will not be sent.
    retryThrottling *retryThrottlingPolicy
    // healthCheckConfig must be set as one of the requirement to enable LB channel
    // health check.
    healthCheckConfig *healthCheckConfig
    // rawJSONString stores service config json string that get parsed into
    // this service config struct.
    rawJSONString string
}

// healthCheckConfig defines the go-native version of the LB channel health check config.
type healthCheckConfig struct {
    // serviceName is the service name to use in the health-checking request.
    ServiceName string
}
```

`parseServiceConfig()` is the main function which is called by gRPC core and resolver. In `parseServiceConfig()`,

- `jsonSC` is used as the intermediate type to parse `ServiceConfig`.
- The `LoadBalancingConfig` field in `jsonSC` is `internalserviceconfig.BalancerConfig`, which is a special type to parse JSON data.
- Note the name of `LoadBalancingConfig` field, That's why I say the `lbConfig` is actually the `LoadBalancingConfig`
- When `err := json.Unmarshal([]byte(js), &rsc)` is called, `BalancerConfig.UnmarshalJSON()` is called for `internalserviceconfig.BalancerConfig` type.

```go
// TODO(lyuxuan): delete this struct after cleaning up old service config implementation.
type jsonMC struct {
    Name                    *[]jsonName
    WaitForReady            *bool
    Timeout                 *string
    MaxRequestMessageBytes  *int64
    MaxResponseMessageBytes *int64
    RetryPolicy             *jsonRetryPolicy
}

// TODO(lyuxuan): delete this struct after cleaning up old service config implementation.
type jsonSC struct {
    LoadBalancingPolicy *string
    LoadBalancingConfig *internalserviceconfig.BalancerConfig
    MethodConfig        *[]jsonMC
    RetryThrottling     *retryThrottlingPolicy
    HealthCheckConfig   *healthCheckConfig
}

func parseServiceConfig(js string) *serviceconfig.ParseResult {
    if len(js) == 0 {
        return &serviceconfig.ParseResult{Err: fmt.Errorf("no JSON service config provided")}
    }
    var rsc jsonSC
    err := json.Unmarshal([]byte(js), &rsc)
    if err != nil {
        logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
        return &serviceconfig.ParseResult{Err: err}
    }
    sc := ServiceConfig{
        LB:                rsc.LoadBalancingPolicy,
        Methods:           make(map[string]MethodConfig),
        retryThrottling:   rsc.RetryThrottling,
        healthCheckConfig: rsc.HealthCheckConfig,
        rawJSONString:     js,
    }
    if c := rsc.LoadBalancingConfig; c != nil {
        sc.lbConfig = &lbConfig{
            name: c.Name,
            cfg:  c.Config,
        }
    }

    if rsc.MethodConfig == nil {
        return &serviceconfig.ParseResult{Config: &sc}
    }

    paths := map[string]struct{}{}
    for _, m := range *rsc.MethodConfig {
        if m.Name == nil {
            continue
        }
        d, err := parseDuration(m.Timeout)
        if err != nil {
            logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
            return &serviceconfig.ParseResult{Err: err}
        }

        mc := MethodConfig{
            WaitForReady: m.WaitForReady,
            Timeout:      d,
        }
        if mc.RetryPolicy, err = convertRetryPolicy(m.RetryPolicy); err != nil {
            logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to %v", js, err)
            return &serviceconfig.ParseResult{Err: err}
        }
        if m.MaxRequestMessageBytes != nil {
            if *m.MaxRequestMessageBytes > int64(maxInt) {
                mc.MaxReqSize = newInt(maxInt)
            } else {
                mc.MaxReqSize = newInt(int(*m.MaxRequestMessageBytes))
            }
        }
        if m.MaxResponseMessageBytes != nil {
            if *m.MaxResponseMessageBytes > int64(maxInt) {
                mc.MaxRespSize = newInt(maxInt)
            } else {
                mc.MaxRespSize = newInt(int(*m.MaxResponseMessageBytes))
            }
        }
        for i, n := range *m.Name {
            path, err := n.generatePath()
            if err != nil {
                logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to methodConfig[%d]: %v", js, i, err)
                return &serviceconfig.ParseResult{Err: err}
            }

            if _, ok := paths[path]; ok {
                err = errDuplicatedName
                logger.Warningf("grpc: parseServiceConfig error unmarshaling %s due to methodConfig[%d]: %v", js, i, err)
                return &serviceconfig.ParseResult{Err: err}
            }
            paths[path] = struct{}{}
            sc.Methods[path] = mc
        }
    }

    if sc.retryThrottling != nil {
        if mt := sc.retryThrottling.MaxTokens; mt <= 0 || mt > 1000 {
            return &serviceconfig.ParseResult{Err: fmt.Errorf("invalid retry throttling config: maxTokens (%v) out of range (0, 1000]", mt)}
        }
        if tr := sc.retryThrottling.TokenRatio; tr <= 0 {
            return &serviceconfig.ParseResult{Err: fmt.Errorf("invalid retry throttling config: tokenRatio (%v) may not be negative", tr)}
        }
    }
    return &serviceconfig.ParseResult{Config: &sc}
}
```

`BalancerConfig.UnmarshalJSON` checks the list of configuration. Each configuration is a pair of `name` and `jsonCfg`.

- `UnmarshalJSON()` uses `name` to find the registered balancer builder.
- The registered builder has to implement `balancer.ConfigParser` interface. `ConfigParser` interface has a `ParseConfig()` method.
- `UnmarshalJSON()` calls the builder's `parser.ParseConfig(jsonCfg)` to parse the `jsonCfg`.
- In our discussion, the value of `name` is `xds_cluster_manager_experimental`.
- The registered balancer builder for `xds_cluster_manager_experimental` is `bal`: cluster manager builder.
- The `ParseConfig()` method of cluster manager builder calls `parseConfig()` function to return the `lbConfig` type.
- Note the return type of `ParseConfig()` is `serviceconfig.LoadBalancingConfig`. `lbConfig` struct embedding `serviceconfig.LoadBalancingConfig` interface. Which means `lbConfig` struct is a `serviceconfig.LoadBalancingConfig` type.

```go
// BalancerConfig wraps the name and config associated with one load balancing
// policy. It corresponds to a single entry of the loadBalancingConfig field
// from ServiceConfig.
//
// It implements the json.Unmarshaler interface.
//
// https://github.com/grpc/grpc-proto/blob/54713b1e8bc6ed2d4f25fb4dff527842150b91b2/grpc/service_config/service_config.proto#L247
type BalancerConfig struct {
    Name   string
    Config externalserviceconfig.LoadBalancingConfig
}

// UnmarshalJSON implements the json.Unmarshaler interface.
//
// ServiceConfig contains a list of loadBalancingConfigs, each with a name and
// config. This method iterates through that list in order, and stops at the
// first policy that is supported.
// - If the config for the first supported policy is invalid, the whole service
//   config is invalid.
// - If the list doesn't contain any supported policy, the whole service config
//   is invalid.
func (bc *BalancerConfig) UnmarshalJSON(b []byte) error {
    var ir intermediateBalancerConfig
    err := json.Unmarshal(b, &ir)
    if err != nil {
        return err
    }

    for i, lbcfg := range ir {
        if len(lbcfg) != 1 {
            return fmt.Errorf("invalid loadBalancingConfig: entry %v does not contain exactly 1 policy/config pair: %q", i, lbcfg)
        }

        var (
            name    string
            jsonCfg json.RawMessage
        )
        // Get the key:value pair from the map. We have already made sure that
        // the map contains a single entry.
        for name, jsonCfg = range lbcfg {
        }

        builder := balancer.Get(name)
        if builder == nil {
            // If the balancer is not registered, move on to the next config.
            // This is not an error.
            continue
        }
        bc.Name = name

        parser, ok := builder.(balancer.ConfigParser)
        if !ok {
            if string(jsonCfg) != "{}" {
                logger.Warningf("non-empty balancer configuration %q, but balancer does not implement ParseConfig", string(jsonCfg))
            }
            // Stop at this, though the builder doesn't support parsing config.
            return nil
        }

        cfg, err := parser.ParseConfig(jsonCfg)
        if err != nil {
            return fmt.Errorf("error parsing loadBalancingConfig for policy %q: %v", name, err)
        }
        bc.Config = cfg
        return nil
    }
    // This is reached when the for loop iterates over all entries, but didn't
    // return. This means we had a loadBalancingConfig slice but did not
    // encounter a registered policy. The config is considered invalid in this
    // case.
    return fmt.Errorf("invalid loadBalancingConfig: no supported policies found")
}

// ConfigParser parses load balancer configs.
type ConfigParser interface {
    // ParseConfig parses the JSON load balancer config provided into an
    // internal form or returns an error if the config is invalid.  For future
    // compatibility reasons, unknown fields in the config should be ignored.
    ParseConfig(LoadBalancingConfigJSON json.RawMessage) (serviceconfig.LoadBalancingConfig, error)
}

const balancerName = "xds_cluster_manager_experimental"

func init() {
    balancer.Register(builder{})
}

type builder struct{}

func (builder) Build(cc balancer.ClientConn, _ balancer.BuildOptions) balancer.Balancer {
    b := &bal{}
    b.logger = prefixLogger(b)
    b.stateAggregator = newBalancerStateAggregator(cc, b.logger)
    b.stateAggregator.start()
    b.bg = balancergroup.New(cc, b.stateAggregator, nil, b.logger)
    b.bg.Start()
    b.logger.Infof("Created")
    return b
}

func (builder) Name() string {
    return balancerName
}

func (builder) ParseConfig(c json.RawMessage) (serviceconfig.LoadBalancingConfig, error) {
    return parseConfig(c)
}

// Config represents an opaque data structure holding a service config.
type Config interface {
    isServiceConfig()
}

// LoadBalancingConfig represents an opaque data structure holding a load
// balancing config.
type LoadBalancingConfig interface {
    isLoadBalancingConfig()
}

// ParseResult contains a service config or an error.  Exactly one must be
// non-nil.
type ParseResult struct {
    Config Config
    Err    error
}

```

The  `parseConfig()` function use the `lbConfig` as the target type to parse the JSON data.

- The `Children` field of `lbConfig` is `map[string]childConfig`. `json.Unmarshal()` can process it as usual.
- The `ChildPolicy` field of `childConfig` is `internalserviceconfig.BalancerConfig`, which is the same `BalancerConfig` as the `LoadBalancingConfig` field in `jsonSC`.
- Remember `BalancerConfig` has a `UnmarshalJSON()` method? `UnmarshalJSON()` will be called again for the `ChildPolicy` field.
- This time the registered balancer builder name can be either `cds_experimental` or `weighted_target_experimental`.
- Each of these builders has their own `ParseConfig()` method.

```go

type childConfig struct {
    // ChildPolicy is the child policy and it's config.
    ChildPolicy *internalserviceconfig.BalancerConfig
}

// lbConfig is the balancer config for xds routing policy.
type lbConfig struct {
    serviceconfig.LoadBalancingConfig
    Children map[string]childConfig
}

func parseConfig(c json.RawMessage) (*lbConfig, error) {
    cfg := &lbConfig{}
    if err := json.Unmarshal(c, cfg); err != nil {
        return nil, err
    }

    return cfg, nil
}
```

```json
{
  "loadBalancingConfig": [
    {
      "xds_cluster_manager_experimental": {
        "children": {
          "cds:cluster_1": {
            "childPolicy": [
              {
                "cds_experimental": {
                  "cluster": "cluster_1"
                }
              }
            ]
          },
          "weighted:cluster_1_cluster_2_1": {
            "childPolicy": [
              {
                "weighted_target_experimental": {
                  "targets": {
                    "cluster_1": {
                      "weight": 75,
                      "childPolicy": [
                        {
                          "cds_experimental": {
                            "cluster": "cluster_1"
                          }
                        }
                      ]
                    },
                    "cluster_2": {
                      "weight": 25,
                      "childPolicy": [
                        {
                          "cds_experimental": {
                            "cluster": "cluster_2"
                          }
                        } 
                      ]
                    }
                  }
                }
              }
            ]
          },
          "weighted:cluster_1_cluster_3_1": {
            "childPolicy": [
              {
                "weighted_target_experimental": {
                  "targets": {
                    "cluster_1": {
                      "weight": 99,
                      "childPolicy": [
                        {
                          "cds_experimental": {
                            "cluster": "cluster_1"
                          }
                        }
                      ]
                    },
                    "cluster_3": {
                      "weight": 1,
                      "childPolicy": [
                        {
                          "cds_experimental": {
                            "cluster": "cluster_3"
                          }
                        }
                      ]
                    }
                  }
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

- CDS balancer has a `ParseConfig()` method. Which uses its own `lbConfig` to parse the JSON data.
- The `ClusterName` field of `lbConfig` is the name of the CDS cluster.
- EDS balancer and weighted target balancer also use the same technique to parse it's own JSON data.

```go
const (
    cdsName = "cds_experimental"
    edsName = "eds_experimental"
)

+-- 19 lines: var (····································································································································
func init() {
    balancer.Register(cdsBB{})
}

// cdsBB (short for cdsBalancerBuilder) implements the balancer.Builder
// interface to help build a cdsBalancer.
// It also implements the balancer.ConfigParser interface to help parse the
// JSON service config, to be passed to the cdsBalancer.
type cdsBB struct{}

// Build creates a new CDS balancer with the ClientConn.
func (cdsBB) Build(cc balancer.ClientConn, opts balancer.BuildOptions) balancer.Balancer {
+-- 35 lines: b := &cdsBalancer{·······················································································································
}

// Name returns the name of balancers built by this builder.
func (cdsBB) Name() string {
    return cdsName
}

// lbConfig represents the loadBalancingConfig section of the service config
// for the cdsBalancer.
type lbConfig struct {
    serviceconfig.LoadBalancingConfig
    ClusterName string `json:"Cluster"`
}

// ParseConfig parses the JSON load balancer config provided into an
// internal form or returns an error if the config is invalid.
func (cdsBB) ParseConfig(c json.RawMessage) (serviceconfig.LoadBalancingConfig, error) {
    var cfg lbConfig
    if err := json.Unmarshal(c, &cfg); err != nil {
        return nil, fmt.Errorf("xds: unable to unmarshal lbconfig: %s, error: %v", string(c), err)
    }
    return &cfg, nil
}

```
