java-prometheus-metrics-agent
=============================

A javaagent for scraping and exposing MBeans to Prometheus

Features
--------

- By default, this agent exposes all the MBeans available.
- Scraping behavior can be customized using jq scripts.

Installation
------------

#### Build from source

Clone this repository and run `mvn clean package`. A agent jar will be built and appear at `target/java-prometheus-metrics-agent-{version}.jar`.

#### Download from Maven Central

TBD

Usage
-----

Add `-javaagent` option to JVM arguments.

```sh
java -javaagent:<PATH_TO_AGENT_JAR> ...
```

Configurations can be passed as a javaagent argument.

```sh
# Pass YAML (or JSON) directly
java -javaagent:<PATH_TO_AGENT_JAR>=<CONFIG_YAML> ...

# Load configurations from PATH_TO_CONFIG_YAML
java -javaagent:<PATH_TO_AGENT_JAR>=@<PATH_TO_CONFIG_YAML> ...
```

Configuration
-------------

### Example

```yaml
# You can omit `server` object if you use default `bind_address`.
server:
    bind_address: '0.0.0.0:18090'
rules:
  - pattern:
      - 'com\\.sun\\.management:type=HotSpotDiagnostic:DiagnosticOptions'
      - 'java\\.lang:type=Threading:AllThreadIds'
    # Drop less useful attributes JVM exposes.
    skip: true
  - pattern:
      - '::.*MinuteRate'
      - '::MeanRate'
    # Some instrumentation libraries (such as Dropwizard Metrics) expose pre-calculated rate statistics.
    # Since Prometheus can calculate rates itself, just drop them.
    skip: true
  - pattern: 'java.lang:type=GarbageCollector:LastGcInfo'
    # NOTE: You probably don't need to do this; Just a demo.
    # For MBean attributes in `java.lang` domain, put the value of the `type` key property and
    # the attribute name in Prometheus metric names, separated by colons. Also, rename `memoryUsageAfterGc_key`
    # and `memoryUsageBeforeGc_key` labels auto-generated from TabularData to `heap`.
    transform: |
        default_transform_v1(["type"]; true; {memoryUsageAfterGc_key: "heap", memoryUsageBeforeGc_key: "heap"})
  - pattern: 'java.lang'
    # NOTE: You probably don't need to do this; Just a demo.
    # For MBean attributes in `java.lang` domain, put the value of the `type` key property and
    # the attribute name in Prometheus metric names, separated by colons.
    transform: |
        default_transform_v1(["type"]; true)
  - pattern: 'procfs'
    # Default transform is default_transform_v1, so this rule is effectively a no-op. Just for a demo purpose.
    transform: default_transform_v1
```

This YAML is mapped to [Config](src/main/java/net/thisptr/java/prometheus/metrics/agent/Config.java) class using Jackson data-binding and validated by Hibernate validator.

### Server Configuration

| Key | Default | Description |
|-|-|-|
| `server.bind_address` | `0.0.0.0:18090` | IP and port which this javaagent listens on. |

### Rule Configuration

Rules are searched in order and a first match is used for each attribute.

| Key | Default | Description |
|-|-|-|
| `rules[].pattern` | `null` | A pattern used to match MBean attributes this rule applies to. A rule with a `null` pattern applies to any attributes. See [Pattern Format](#pattern-format) for syntax details. |
| `rules[].skip` | `false` | If `true`, skip exposition of the attribute to Prometheus. |
| `rules[].transform` | `default_transform_v1` | A jq script to convert a MBean attribute to Prometheus metrics. See [Transform Script](#transform-script) for details. |

#### Pattern Format

TBD

#### Transform Script

TBD

##### Using generic `default_transform_v1` function

Writing a generic `transform` script from scratch is hard because MBean attributes contain variety of types including a nested ones such as CompositeData, TabularData, etc.
Thus, we provide the following `transform` implementations.

- default_transform_v1 ([prometheus.jq#L44](src/main/resources/prometheus.jq#44))

  Same as `default_transform_v1([]; false; {})`.

- default_transform_v1(List&lt;String&gt; name_keys, boolean attribute_as_metric_name) ([prometheus.jq#L30](src/main/resources/prometheus.jq#30))

  Same as `default_transform_v1(name_keys; attribute_as_metric_name; {})`

- default_transform_v1(List&lt;String&gt; name_keys, boolean attribute_as_metric_name, Map&lt;String, String&gt; label_remapping) ([prometheus.jq#L45](src/main/resources/prometheus.jq#45))

Given the following MBean attribute JSON,

```javascript
{
  "type": "javax.management.openmbean.CompositeData",
  "value": {
    "$type": "javax.management.openmbean.CompositeData",
    "committed": 99745792,
    "init": 0,
    "max": 134217728,
    "used": 95750352
  },
  "domain": "java.lang",
  "properties": {
    "type": "MemoryPool",
    "name": "Metaspace"
  },
  "attribute": "Usage"
}
```

1. `default_transform_v1` produces

   ```javascript
   {"name":"java.lang","labels":{"type":"MemoryPool","name":"Metaspace","attribute":"Usage_committed"},"value":99745792}
   {"name":"java.lang","labels":{"type":"MemoryPool","name":"Metaspace","attribute":"Usage_init"},"value":0}
   {"name":"java.lang","labels":{"type":"MemoryPool","name":"Metaspace","attribute":"Usage_max"},"value":134217728}
   {"name":"java.lang","labels":{"type":"MemoryPool","name":"Metaspace","attribute":"Usage_used"},"value":95750352}
   ```
   
   which corresponds to the following lines in [Prometheus exposition format](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md).
   
   ```text
   java_lang{type="MemoryPool", name="Metaspace", attribute="Usage_committed", } 99745792.0
   java_lang{type="MemoryPool", name="Metaspace", attribute="Usage_init", } 0.0
   java_lang{type="MemoryPool", name="Metaspace", attribute="Usage_max", } 134217728.0
   java_lang{type="MemoryPool", name="Metaspace", attribute="Usage_used", } 95750352.0
   ```
   
   Note that illegal characters in metric name and label names are automatically replaced by `_`. See [Data Model](https://prometheus.io/docs/concepts/data_model/) in the Prometheus documentation for metrics naming.

2. `default_transform_v1(["type"]; false)`
   
   ```text
   java_lang:MemoryPool{name="Metaspace", attribute="Usage_committed", } 99745792.0
   java_lang:MemoryPool{name="Metaspace", attribute="Usage_init", } 0.0
   java_lang:MemoryPool{name="Metaspace", attribute="Usage_max", } 134217728.0
   java_lang:MemoryPool{name="Metaspace", attribute="Usage_used", } 95750352.0
   ```

3. `default_transform_v1(["type"]; true)`
   
   ```text
   java_lang:MemoryPool:Usage_committed{name="Metaspace", } 99745792.0
   java_lang:MemoryPool:Usage_init{name="Metaspace", } 0.0
   java_lang:MemoryPool:Usage_max{name="Metaspace", } 134217728.0
   java_lang:MemoryPool:Usage_used{name="Metaspace", } 95750352.0
   ```

4. `default_transform_v1(["type"]; true, {"name": "heapname"})`
   
   ```text
   java_lang:MemoryPool:Usage_committed{heapname="Metaspace", } 99745792.0
   java_lang:MemoryPool:Usage_init{heapname="Metaspace", } 0.0
   java_lang:MemoryPool:Usage_max{heapname="Metaspace", } 134217728.0
   java_lang:MemoryPool:Usage_used{heapname="Metaspace", } 95750352.0
   ```

### Debugging

Sometimes it's hard to debug complex `transform` scripts. Here's some tips and tricks to debug them.

#### Fetching all MBeans as JSON

You can navigate to `http://<HOST>:<PORT>/mbeans` on your browser and it will print all the MBeans in JSON.
These JSONs can be fed into your `transform` script using jq (see below).

#### Using jq to test `transform` script

Pre-defined jq functions can be found at [src/main/resources/prometheus.jq](src/main/resources/prometheus.jq). You can include it in jq to test your `transform` script. E.g.

```sh
jq -c 'include "src/main/resources/prometheus"; default_transform_v1'
```

#### Changing a log level to FINEST

This agent uses JUL framework for logging. Errors caused by user configurations are logged at &gt;= INFO level. There are still some errors
logged at &lt; INFO (such as FINEST level). Consider setting the log level to FINEST, especially if you are silently missing some attributes.

To change the log level, create your `logging.properties` and set its path to `java.util.logging.config.file` system property.

```console
$ cat logging.properties
handlers = java.util.logging.ConsoleHandler
.level = INFO
java.util.logging.ConsoleHandler.level = FINEST
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
java.util.logging.SimpleFormatter.format = %1$tFT%1$tT.%1$tL %4$-5.5s %3$-80.80s : %5$s%6$s%n
net.thisptr.java.prometheus.metrics.agent.level = FINEST
net.thisptr.java.prometheus.metrics.agent.shade.level = INFO

$ java -Djava.util.logging.config.file=logging.properties ...
```

### Real-world Examples

TBD

References
----------

 - [Java Management Extensions (JMX) - Best Practices](http://www.oracle.com/technetwork/articles/java/best-practices-jsp-136021.html)

License
-------

The MIT License.