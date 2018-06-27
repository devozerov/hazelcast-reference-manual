
### Synchronizing WAN Target Cluster

Starting with Hazelcast 3.7 you can initiate a synchronization operation on an IMap for a specific target cluster. 
Synchronization operation sends all the data of an IMap to a target cluster to align the state of target IMap with source IMap.
Synchronization is useful if two remote clusters lost their synchronization due to WAN queue overflow or in restart scenarios.

Synchronization can be initiated through [Management Center](http://docs.hazelcast.org/docs/management-center/latest/manual/html/index.html#wan-sync) and Hazelcast’s REST API.

Below is the URL for the REST call;

```
http://member_ip:port/hazelcast/rest/mancenter/wan/sync/map
```

You need to add parameters to the request in the following order separated by "&";

  - Name of the WAN replication configuration
  - Target group name
  - Map name to be synchronized


Additionally, you can specify a time range in which an entry must have been updated to be replicated to the target cluster. The range is specified by the "from" and "to" timestamps, which reflect the time as provided by the configured Hazelcast clock -`com.hazelcast.util.Clock#currentTimeMillis`. This clock is the system clock by default - `java.lang.System.currentTimeMillis`.
 


Assume that you have configured an IMap with a WAN replication configuration as follows:

```xml
<wan-replication name="my-wan-cluster">
      <wan-publisher group-name="istanbul">
          <class-name>com.hazelcast.enterprise.wan.replication.WanBatchReplication</class-name>
            ...
      </wan-publisher>
<wan-replication>
...
<map name="my-map">
    <wan-replication-ref name="my-wan-cluster">
       <merge-policy>com.hazelcast.map.merge.PassThroughMergePolicy</merge-policy>
    </wan-replication-ref>
</map>
```

Then, a sample CURL command to initiate the synchronization for "my-map" would be as follows:

```
curl -H "Content-type: text/plain" -X POST -d "my-wan-cluster&istanbul&my-map" --URL http://127.0.0.1:5701/hazelcast/rest/mancenter/wan/sync/map
```

If you would like to specify the time range in which the entries must have been modified to be replicated, you may add two additional parameters, the "from" and "to" timestamps, as shown below:
 
```
curl -H "Content-type: text/plain" -X POST -d "my-wan-cluster&istanbul&my-map&1528392958409&1528392958900" --URL http://127.0.0.1:5701/hazelcast/rest/mancenter/wan/sync/map
```
 
If you would like to specify just the "from" timestamp, you can set the "to" timestamp to `9223372036854775807` (`Long.MAX_VALUE`). If you would like to specify just the "to" timestamp, you can set the "from" timestamp to `0`.
 


##### Synchronizing All Maps
 
You can also use the following URL in your REST call if you want to synchronize all the maps in source and target cluster:
 
`http://member_ip:port/hazelcast/rest/mancenter/wan/sync/allmaps`
 
As explained above, you can also specify the time range in which the entries must have been modified to be replicated, as shown below:
 
```
curl -H "Content-type: text/plain" -X POST -d "my-wan-cluster&istanbul&1528392958409&1528392958900" --URL http://127.0.0.1:5701/hazelcast/rest/mancenter/wan/sync/allmaps
```
 
Again, you can also provide only "to" or "from" timestamps, as described above.
