# xDS protocol - CDS/EDS

- [Wrappers for balancers](#wrappers-for-balancers)
  - [Resolver](#1)
  - [Cluster Manager](#2)
  - [CDS balancer](#3)
  - [EDS balancer](#4)
  - [Priority locality](#5)
- [Which balancer, which `ClientConn` ?](#which-balancer-which-clientconn-)

## Wrappers for balancers

### Which balancer, which `ClientConn` ?

Before we continue the discussion of `acBalancerWrapper.Connect()`, let's find out some key value of balancer and `ClientConn`/`cc`. If you understand the value of them, it will be easier to understand the following code.

- In `ClientConn.switchBalancer()`, `newCCBalancerWrapper()` is called to create `ccBalancerWrapper`.
  - `ccBalancerWrapper.cc` is of type `ClientConn`.
  - In `newCCBalancerWrapper()`, `b.Build()` uses `ccb` as `cc balancer.ClientConn` parameter, `ccb` is of type `ccBalancerWrapper`.
  - In `newCCBalancerWrapper()`, `b.Build()` is actually `builder.Build()`. It's the balancer cluster manager.
    - In `builder.Build()`, `b.bg` is of type `BalancerGroup`.
    - In `balancergroup.New()`, `b.bg.cc` is of type `ccBalancerWrapper`.

```go
func newCCBalancerWrapper(cc *ClientConn, b balancer.Builder, bopts balancer.BuildOptions) *ccBalancerWrapper {
    ccb := &ccBalancerWrapper{
        cc:       cc,
        scBuffer: buffer.NewUnbounded(),
        done:     grpcsync.NewEvent(),
        subConns: make(map[*acBalancerWrapper]struct{}),
    }
    go ccb.watcher()
    ccb.balancer = b.Build(ccb, bopts)
    return ccb
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

// New creates a new BalancerGroup. Note that the BalancerGroup
// needs to be started to work.
func New(cc balancer.ClientConn, stateAggregator BalancerStateAggregator, loadStore load.PerClusterReporter, logger *grpclog.PrefixLogger) *BalancerGroup {
    return &BalancerGroup{
        cc:        cc,
        logger:    logger,
        loadStore: loadStore,

        stateAggregator: stateAggregator,

        idToBalancerConfig: make(map[string]*subBalancerWrapper),
        balancerCache:      cache.NewTimeoutCache(DefaultSubBalancerCloseTimeout),
        scToSubBalancer:    make(map[balancer.SubConn]*subBalancerWrapper),
    }
}
```

For CDS balancer, in `bal.updateChildren()`, `b.bg.Add()` is called to add CDS sub-balancers to the group.

- Please note there exists two balancer group. CDS balancer group and endpoints balancer group.
- The CDS balancer builder is get from the `lbConfig`. Checks `balancer.Get(newT.ChildPolicy.Name)`.

```go
func (b *bal) updateChildren(s balancer.ClientConnState, newConfig *lbConfig) {
    update := false
    addressesSplit := hierarchy.Group(s.ResolverState.Addresses)

    // Remove sub-pickers and sub-balancers that are not in the new cluster list.
    for name := range b.children {
        if _, ok := newConfig.Children[name]; !ok {
            b.stateAggregator.remove(name)
            b.bg.Remove(name)
            update = true
        }
    }

    // For sub-balancers in the new cluster list,
    // - add to balancer group if it's new,
    // - forward the address/balancer config update.
    for name, newT := range newConfig.Children {
        if _, ok := b.children[name]; !ok {
            // If this is a new sub-balancer, add it to the picker map.
            b.stateAggregator.add(name)
            // Then add to the balancer group.
            b.bg.Add(name, balancer.Get(newT.ChildPolicy.Name))
        }
        // TODO: handle error? How to aggregate errors and return?
        _ = b.bg.UpdateClientConnState(name, balancer.ClientConnState{
            ResolverState: resolver.State{
                Addresses:     addressesSplit[name],
                ServiceConfig: s.ResolverState.ServiceConfig,
                Attributes:    s.ResolverState.Attributes,
            },
            BalancerConfig: newT.ChildPolicy.Config,
        })
    }

    b.children = newConfig.Children
    if update {
        b.stateAggregator.buildAndUpdate()
    }
}
```

In `BalancerGroup.Add()`, `subBalancerWrapper` is created with `subBalancerWrapper.ClientConn` field set to `bg.cc`.

- From the previous code, `bg.cc` is of type `ccBalancerWrapper`.
- `Add()` calls `sbc.startBalancer()` to create CDS balancer.
- `subBalancerWrapper.ClientConn` is `ccBalancerWrapper`

```go
// Add adds a balancer built by builder to the group, with given id.
func (bg *BalancerGroup) Add(id string, builder balancer.Builder) {
    // Store data in static map, and then check to see if bg is started.
    bg.outgoingMu.Lock()
    var sbc *subBalancerWrapper
    // If outgoingStarted is true, search in the cache. Otherwise, cache is
    // guaranteed to be empty, searching is unnecessary.
    if bg.outgoingStarted {
        if old, ok := bg.balancerCache.Remove(id); ok {
            sbc, _ = old.(*subBalancerWrapper)
            if sbc != nil && sbc.builder != builder {
+-- 12 lines: If the sub-balancer in cache was built with a different······························································································
            }
        }
    }
    if sbc == nil {
        sbc = &subBalancerWrapper{
            ClientConn: bg.cc,
            id:         id,
            group:      bg,
            builder:    builder,
        }
        if bg.outgoingStarted {
            // Only start the balancer if bg is started. Otherwise, we only keep the
            // static data.
            sbc.startBalancer()
        }
    } else {
        // When brining back a sub-balancer from cache, re-send the cached
        // picker and state.
        sbc.updateBalancerStateWithCachedPicker()
    }
    bg.idToBalancerConfig[id] = sbc
    bg.outgoingMu.Unlock()
}
```

In `subBalancerWrapper.startBalancer()`, `sbc` is set to be the argument of `cc ClientConn` parameter.

- Here, `sbc.builder` is `cdsBB`. That is the CDS balancer builder.
- In `cdsBB.Build()`, `ccWrapper` is created and the `ccWrapper.ClientConn` field is set to `subBalancerWrapper`.

```go
func (sbc *subBalancerWrapper) startBalancer() {
    b := sbc.builder.Build(sbc, balancer.BuildOptions{})
    sbc.group.logger.Infof("Created child policy %p of type %v", b, sbc.builder.Name())
    sbc.balancer = b
    if sbc.ccState != nil {
        b.UpdateClientConnState(*sbc.ccState)
    }
}

// cdsBB (short for cdsBalancerBuilder) implements the balancer.Builder
// interface to help build a cdsBalancer.
// It also implements the balancer.ConfigParser interface to help parse the
// JSON service config, to be passed to the cdsBalancer.
type cdsBB struct{}

// Build creates a new CDS balancer with the ClientConn.
func (cdsBB) Build(cc balancer.ClientConn, opts balancer.BuildOptions) balancer.Balancer {
    b := &cdsBalancer{
        bOpts:       opts,
        updateCh:    buffer.NewUnbounded(),
        closed:      grpcsync.NewEvent(),
        cancelWatch: func() {}, // No-op at this point.
        xdsHI:       xdsinternal.NewHandshakeInfo(nil, nil),
    }
    b.logger = prefixLogger((b))
    b.logger.Infof("Created")

    client, err := newXDSClient()
    if err != nil {
        b.logger.Errorf("failed to create xds-client: %v", err)
        return nil
    }
    b.xdsClient = client

    var creds credentials.TransportCredentials
    switch {
    case opts.DialCreds != nil:
        creds = opts.DialCreds
    case opts.CredsBundle != nil:
        creds = opts.CredsBundle.TransportCredentials()
    }
    if xc, ok := creds.(interface{ UsesXDS() bool }); ok && xc.UsesXDS() {
        b.xdsCredsInUse = true
    }
    b.logger.Infof("xDS credentials in use: %v", b.xdsCredsInUse)

    b.ccw = &ccWrapper{
        ClientConn: cc,
        xdsHI:      b.xdsHI,
    }
    go b.run()
    return b
}

// Name returns the name of balancers built by this builder.
func (cdsBB) Name() string {
    return cdsName
}

```
