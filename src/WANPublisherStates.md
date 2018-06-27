
### WAN Publisher States

WAN publishers can be in different states:

- `REPLICATING` (default): State where both enqueuing new events is allowed, enqueued events are replicated to the target cluster and WAN synchronization is enabled.
- `PAUSED`: State where new events are enqueued but they are dequeued. Some events which have been dequeued before the state was switched may still be replicated to the target cluster; but further events will not be replicated. WAN synchronization is enabled.
- `STOPPED`: State where neither new events are enqueued nor dequeued. As with the `PAUSED` state, some events might still be replicated after the publisher has switched to this state. WAN synchronization is enabled.
 
Because WAN events are enqueued in the `PAUSED` state, WAN queues may fill up and start dropping events. This means that a WAN publisher will always consume memory for the WAN queues, even if the target cluster will no longer be available. 

In the case when you would like to stop replicating to a WAN publisher indefinitely, it is better to use the `STOPPED` state. If you have switched the publisher state to `STOPPED`, you might additionally consider clearing the WAN queues for that publisher, to release any memory used by the already enqueued events.
 
In addition, it is possible to define the initial WAN publisher state, using the configuration element `initial-publisher-state`. This state will immediately take effect when the member is starting up. This may be useful when you are starting with some WAN publishers which are not currently available but may become available in the future. 

An example of a declarative configuration with a WAN publisher in the initial state of `STOPPED` is shown below:
 
 
```
<hazelcast>
...
  <wan-replication name="my-wan-cluster-batch">
      <wan-publisher group-name="london">
          <class-name>com.hazelcast.enterprise.wan.replication.WanBatchReplication</class-name>
          <initial-publisher-state>STOPPED</initial-publisher-state>
          <properties>
              ...
          </properties>
      </wan-publisher>
  </wan-replication>
...
</hazelcast>
```
 
An analogous example can be made using programmatic configuration.
 
In cases where you know there may be some future WAN publishers but do not yet know how many or what the target endpoints may be, you may define any number of WAN publishers using a custom discovery SPI and in the initial state of `STOPPED`. Your discovery SPI implementation may return target endpoint addresses from some location which is under your control (like a configuration registry or a file). Once you want to start replicating to a new WAN publisher, you can update the list of target endpoints and switch the publisher state to `REPLICATING`. Please see the [Defining WAN Replication Using Discovery SPI section](#defining-wan-replication-using-discovery-spi).
 
You may also switch the WAN publisher states using [Hazelcast Management Center](http://docs.hazelcast.org/docs/management-center/latest/manual/html/index.html#changing-wan-publisher-state) while the cluster is already running.