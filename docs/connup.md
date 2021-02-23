
# Connect to Upstream server

In `handleEDSResponse()`, `edsImpl.handleEDSResponsePerPriority()` is called to initialize connection?

```go
    for priority, newLocalities := range newLocalitiesWithPriority {
        if !priorityLowest.isSet() || priorityLowest.higherThan(priority) {
            priorityLowest = priority
        }

        bgwc, ok := edsImpl.priorityToLocalities[priority]
        if !ok {
            // Create balancer group if it's never created (this is the first
            // time this priority is received). We don't start it here. It may
            // be started when necessary (e.g. when higher is down, or if it's a
            // new lowest priority).
            ccPriorityWrapper := edsImpl.ccWrapperWithPriority(priority)
            stateAggregator := weightedaggregator.New(ccPriorityWrapper, edsImpl.logger, newRandomWRR)
            bgwc = &balancerGroupWithConfig{
                bg:              balancergroup.New(ccPriorityWrapper, stateAggregator, edsImpl.loadReporter, edsImpl.logger),
                stateAggregator: stateAggregator,
                configs:         make(map[internal.LocalityID]*localityConfig),
            }
            edsImpl.priorityToLocalities[priority] = bgwc
            priorityChanged = true
            edsImpl.logger.Infof("New priority %v added", priority)
        }
        edsImpl.handleEDSResponsePerPriority(bgwc, newLocalities)
    }
```

In `handleEDSResponsePerPriority()`, `bgwc.bg.UpdateClientConnState()` is called to forward the request to `bgwc.bg`.

- Here, `bgwc` is the `balancerGroupWithConfig` parameter passed to `handleEDSResponsePerPriority()`.
- `bgwc` is created in `handleEDSResponse()`.
- `bgwc.bg` is also created in `handleEDSResponse()`.

```go
        if addrsChanged {
            config.addrs = newAddrs
            bgwc.bg.UpdateClientConnState(lidJSON, balancer.ClientConnState{
                ResolverState: resolver.State{Addresses: newAddrs},
            })
        }
```

In `BalancerGroup.UpdateClientConnState()`, `bg.idToBalancerConfig[id]` is checked, if `id` exists, then calls `config.updateClientConnState()`.

- Here, `config` is of type `*subBalancerWrapper`.
- `bg.idToBalancerConfig[id]` is created in `BalancerGroup.Add()`.
- `config.updateClientConnState()` is actually `subBalancerWrapper.updateClientConnState()`.
- Before we jump into `subBalancerWrapper.updateClientConnState()`, Let's figure out some key value.

```go
// UpdateClientConnState handles ClientState (including balancer config and
// addresses) from resolver. It finds the balancer and forwards the update.
func (bg *BalancerGroup) UpdateClientConnState(id string, s balancer.ClientConnState) error {
    bg.outgoingMu.Lock()
    defer bg.outgoingMu.Unlock()
    if config, ok := bg.idToBalancerConfig[id]; ok {
        return config.updateClientConnState(s)
    }
    return nil
}
```

`edsImpl.subBalancerBuilder` is initialized when `edsImpl` is created. It get the value from `balancer.Get(roundrobin.Name)`: the round-robin balancer builder.

- `balancer.Get(roundrobin.Name)` returns `baseBuilder` with `name` parameter as `"round_robin"`
- That's why it is a round-robin balancer builder.

```go
// newEDSBalancerImpl create a new edsBalancerImpl.
func newEDSBalancerImpl(cc balancer.ClientConn, enqueueState func(priorityType, balancer.State), lr load.PerClusterReporter, logger *grpclog.PrefixLogger) *edsBalancerImpl {
    edsImpl := &edsBalancerImpl{
        cc:                 cc,
        logger:             logger,
        subBalancerBuilder: balancer.Get(roundrobin.Name),
        loadReporter:       lr,

        enqueueChildBalancerStateUpdate: enqueueState,

        priorityToLocalities: make(map[priorityType]*balancerGroupWithConfig),
        priorityToState:      make(map[priorityType]*balancer.State),
        subConnToPriority:    make(map[balancer.SubConn]priorityType),
    }
    // Don't start balancer group here. Start it when handling the first EDS
    // response. Otherwise the balancer group will be started with round-robin,
    // and if users specify a different sub-balancer, all balancers in balancer
    // group will be closed and recreated when sub-balancer update happens.
    return edsImpl
}

// Name is the name of round_robin balancer.
const Name = "round_robin"                                                                        
                               
// newBuilder creates a new roundrobin balancer builder.                                
func newBuilder() balancer.Builder {                                                               
    return base.NewBalancerBuilder(Name, &rrPickerBuilder{}, base.Config{HealthCheck: true})
}                           

func init() {                 
    balancer.Register(newBuilder())                           
}                                                               

// NewBalancerBuilder returns a base balancer builder configured by the provided config.
func NewBalancerBuilder(name string, pb PickerBuilder, config Config) balancer.Builder {
    return &baseBuilder{
        name:          name,
        pickerBuilder: pb,
        config:        config,
    }
}
```

In `handleEDSResponsePerPriority()`, `BalancerGroup.Add()` is called for every `xdsclient.Locality`.

- `edsImpl.subBalancerBuilder` is the argument for the `builder balancer.Builder` parameter.
- `lidJSON` is the JSON representation of `locality.ID`, as the argument for the `id string` parameter.

```go
        if !ok {
            // A new balancer, add it to balancer group and balancer map.
            bgwc.stateAggregator.Add(lidJSON, newWeight)
            bgwc.bg.Add(lidJSON, edsImpl.subBalancerBuilder)
            config = &localityConfig{
                weight: newWeight,
            }
            bgwc.configs[lid] = config

            // weightChanged is false for new locality, because there's no need
            // to update weight in bg.
            addrsChanged = true
            edsImpl.logger.Infof("New locality %v added", lid)
        } else {
            // Compare weight and addrs.
            if config.weight != newWeight {
                weightChanged = true
            }
            if !cmp.Equal(config.addrs, newAddrs) {
                addrsChanged = true
            }
            edsImpl.logger.Infof("Locality %v updated, weightedChanged: %v, addrsChanged: %v", lid, weightChanged, addrsChanged)
        }
```

`BalancerGroup.Add()` calls `sbc.startBalancer()` to initialize the `balancer` field.

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
                // If the sub-balancer in cache was built with a different
                // balancer builder, don't use it, cleanup this old-balancer,
                // and behave as sub-balancer is not found in cache.
                //
                // NOTE that this will also drop the cached addresses for this
                // sub-balancer, which seems to be reasonable.
                sbc.stopBalancer()
                // cleanupSubConns must be done before the new balancer starts,
                // otherwise new SubConns created by the new balancer might be
                // removed by mistake.
                bg.cleanupSubConns(sbc)
                sbc = nil
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

`sbc.startBalancer()` calls `sbc.builder.Build()` to create a balancer. `sbc.builder.Build()` is actually `baseBuilder.Build()`.

- `baseBuilder.Build()` returns `baseBalancer`.
- `baseBalancer` is assigned to `sbc.balancer`.
- Please note that `startBalancer()` uses `sbc` as the `cc ClientConn` parameter.

```go
func (sbc *subBalancerWrapper) startBalancer() {
    b := sbc.builder.Build(sbc, balancer.BuildOptions{})
    sbc.group.logger.Infof("Created child policy %p of type %v", b, sbc.builder.Name())
    sbc.balancer = b
    if sbc.ccState != nil {
        b.UpdateClientConnState(*sbc.ccState)
    }
}

func (bb *baseBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
    bal := &baseBalancer{
        cc:            cc,
        pickerBuilder: bb.pickerBuilder,

        subConns: make(map[resolver.Address]balancer.SubConn),
        scStates: make(map[balancer.SubConn]connectivity.State),
        csEvltr:  &balancer.ConnectivityStateEvaluator{},
        config:   bb.config,
    }
    // Initialize picker to a picker that always returns
    // ErrNoSubConnAvailable, because when state of a SubConn changes, we
    // may call UpdateState with this picker.
    bal.picker = NewErrPicker(balancer.ErrNoSubConnAvailable)
    return bal
}
```

In `updateClientConnState()`,  `sbc.ccState` is set.

- Then `b.UpdateClientConnState()` is called.  which is actually `baseBalancer.UpdateClientConnState()`.
- `baseBalancer` has an `UpdateClientConnState` method.

```go
func (sbc *subBalancerWrapper) updateClientConnState(s balancer.ClientConnState) error {
    sbc.ccState = &s
    b := sbc.balancer
    if b == nil {
        // This sub-balancer was closed. This should never happen because
        // sub-balancers are closed when the locality is removed from EDS, or
        // the balancer group is closed. There should be no further address
        // updates when either of this happened.
        //
        // This will be a common case with priority support, because a
        // sub-balancer (and the whole balancer group) could be closed because
        // it's the lower priority, but it can still get address updates.
        return nil
    }
    return b.UpdateClientConnState(s)
}
```

In `UpdateClientConnState()`, `b.cc.NewSubConn()` is actually `subBalancerWrapper.NewSubConn()`.

- `subBalancerWrapper.NewSubConn()` calls `sbc.group.newSubConn()`.
- `sbc.group.newSubConn()` calls `bg.cc.NewSubConn()`.

```go
func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) error {
    // TODO: handle s.ResolverState.ServiceConfig?
    if logger.V(2) {
        logger.Info("base.baseBalancer: got new ClientConn state: ", s)
    }
    // Successful resolution; clear resolver error and ensure we return nil.
    b.resolverErr = nil
    // addrsSet is the set converted from addrs, it's used for quick lookup of an address.
    addrsSet := make(map[resolver.Address]struct{})
    for _, a := range s.ResolverState.Addresses {
        // Strip attributes from addresses before using them as map keys. So
        // that when two addresses only differ in attributes pointers (but with
        // the same attribute content), they are considered the same address.
        //
        // Note that this doesn't handle the case where the attribute content is
        // different. So if users want to set different attributes to create
        // duplicate connections to the same backend, it doesn't work. This is
        // fine for now, because duplicate is done by setting Metadata today.
        //
        // TODO: read attributes to handle duplicate connections.
        aNoAttrs := a
        aNoAttrs.Attributes = nil
        addrsSet[aNoAttrs] = struct{}{}
        if sc, ok := b.subConns[aNoAttrs]; !ok {
            // a is a new address (not existing in b.subConns).
            //
            // When creating SubConn, the original address with attributes is
            // passed through. So that connection configurations in attributes
            // (like creds) will be used.
            sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})
            if err != nil {
                logger.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)
                continue
            }
            b.subConns[aNoAttrs] = sc
            b.scStates[sc] = connectivity.Idle
            sc.Connect()
        } else {
            // Always update the subconn's address in case the attributes
            // changed.
            //
            // The SubConn does a reflect.DeepEqual of the new and old
            // addresses. So this is a noop if the current address is the same
            // as the old one (including attributes).
            sc.UpdateAddresses([]resolver.Address{a})
        }               
    }           
    for a, sc := range b.subConns {                                      
        // a was removed by resolver.
        if _, ok := addrsSet[a]; !ok {
            b.cc.RemoveSubConn(sc)                                    
            delete(b.subConns, a)                                             
            // Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
            // The entry will be deleted in UpdateSubConnState.
        }
    }
    // If resolver state contains no addresses, return an error so ClientConn
    // will trigger re-resolve. Also records this as an resolver error, so when
    // the overall state turns transient failure, the error message will have
    // the zero address information.
    if len(s.ResolverState.Addresses) == 0 {
        b.ResolverError(errors.New("produced zero addresses"))
        return balancer.ErrBadResolverState
    }
    return nil
}

// NewSubConn overrides balancer.ClientConn, so balancer group can keep track of
// the relation between subconns and sub-balancers.
func (sbc *subBalancerWrapper) NewSubConn(addrs []resolver.Address, opts balancer.NewSubConnOptions) (balancer.SubConn, error) {
    return sbc.group.newSubConn(sbc, addrs, opts)
}

// Following are actions from sub-balancers, forward to ClientConn.

// newSubConn: forward to ClientConn, and also create a map from sc to balancer,
// so state update will find the right balancer.
//
// One note about removing SubConn: only forward to ClientConn, but not delete
// from map. Delete sc from the map only when state changes to Shutdown. Since
// it's just forwarding the action, there's no need for a removeSubConn()
// wrapper function.
func (bg *BalancerGroup) newSubConn(config *subBalancerWrapper, addrs []resolver.Address, opts balancer.NewSubConnOptions) (balancer.SubConn, error) {
    // NOTE: if balancer with id was already removed, this should also return
    // error. But since we call balancer.stopBalancer when removing the balancer, this
    // shouldn't happen.
    bg.incomingMu.Lock()
    if !bg.incomingStarted {
        bg.incomingMu.Unlock()
        return nil, fmt.Errorf("NewSubConn is called after balancer group is closed")
    }
    sc, err := bg.cc.NewSubConn(addrs, opts)
    if err != nil {
        bg.incomingMu.Unlock()
        return nil, err
    }
    bg.scToSubBalancer[sc] = config
    bg.incomingMu.Unlock()
    return sc, nil
}

```
