# Endpoint Discovery

The observers are responsible for discovering service endpoints.  For these
service endpoints to result in a new monitor instance that is watching that
endpoint, you must apply _discovery rules_ to your monitor configuration. Every
monitor that supports monitoring specific services (i.e. not a static monitor
like the `cpu` monitor) can be configured with a `discoveryRule`
config option that specifies a rule using a mini rule language.

For example, to monitor a Redis instance that has been discovered by a
container-based observer, you could use the following configuration:

```yaml
  monitors:
    - type: collectd/redis
      discoveryRule: container_image =~ "redis" && port == 6379
```

## Rule DSL

A rule is an expression that is matched against each discovered endpoint to
determine if a monitor should be active for a particular endpoint. The basic
operators are:

| Operator | Description |
| --- | --- |
| == | Equals |
| != | Not equals |
| < | Less than |
| <= | Less than or equal |
| > | Greater than |
| >= | Greater than or equal |
| =~ | Regex matches |
| !~ | Regex does not match |
| && | And |
| \|\| | Or | 

For all available operators, see <a target="_blank" 
href="https://github.com/antonmedv/expr/blob/master/docs/Language-Definition.md">the
expr language definition</a> (this is what the agent uses under the covers).
We have a shim set of logic that lets you use the `=~` operator even though it
is not actually part of the expr language -- mainly to preserve backwards
compatibility with older agent releases before expr was used.

The variables available in the expression are dependent on which observer you
are using.  The following three variables are common to all observers:

 - `host` (string): The hostname or IP address of the discovered endpoint
 - `port` (integer): The port number of the discovered endpoint
 - `port_type` (`UDP` or `TCP`): Whether the port is TCP or UDP

For a list of observers and the discovery rule variables they provide, see [Observers](./observer-config.md). 

### Additional Functions

In addition, these extra functions are provided:

 - `Get(map, key)` - returns the value from map if the given key is found, otherwise nil

   ```yaml
   discoveryRule: Get(container_labels, "mapKey") == "mapValue"
   ```

   `Get` accepts an optional third argument that will act as a default value in
   the case that the key is not present in the input map.

 - `Contains(map, key)` - returns true if key is inside map, otherwise false

   ```yaml
   discoveryRule: Contains(container_labels, "mapKey")
   ```


There are no implicit rules built into the agent, so each rule must be specified
manually in the config file, in conjunction with the monitor that should monitor the
discovered service.

## Endpoint Config Mapping

Sometimes it might be useful to use certain attributes of a discovered
endpoint (see [Endpoint Discovery](#endpoint-discovery)).  These discovered
endpoints are created by [observers](./observer-config.md) and will usually
contain a full set of metadata that the observer obtains coincidently when it
is doing discovery (e.g. container labels).  This metadata can be mapped
directly to monitor configuration for the monitor that is instantiated for that
endpoint.

To do this, you can set the [configEndpointMappings option](./monitor-config.md)
on a monitor config block.   For example, the `collectd/kafka` monitor has
the `clusterName` config option, which is an arbirary value used to group
together broker instances.  You could derive this from the `cluster` container
label on the kafka container instances like this:

```yaml
monitors:
 - type: collectd/kafka
   discoveryRule: 'container_image =~ "kafka" && port == 9999'
   configEndpointMappings:
     clusterName: 'Get(container_labels, "cluster")'
```

## Troubleshooting

The simplest way to see what services an instance of the agent has discovered
is to run the command `signalfx-agent status endpoints`.  The fields shown will
be the same values that can be used in discovery rules.

## Manually Defined Services

While service discovery is useful, sometimes it is just easier to manually
define services to monitor.  This can be done by setting the `host` and
`port` option in a monitor's config to the host and port that you need to
monitor.  These two values are the core of what the auto-discovery mechanism
provides for you automatically.

For example (making use of YAML references to reduce repetition):

```yaml
  - &es
    type: elasticsearch
    username: admin
    password: s3cr3t
    host: es
    port: 9200
  
  - <<: *es
    host: es2
    port: 9300
```

This would monitor two ElasticSearch instances at the given hosts and ports and would
use the same username and password given in the top-level monitor config to
connect to both of them.  If you needed different configuration for the two ES
hosts, you could simply define two monitor configurations, each with one
service endpoint.

It is invalid to have both manually defined service endpoints and a discovery rule
on a single monitor configuration.
