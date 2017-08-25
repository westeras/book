# Jolokia JMX Reporting

[Jolokia](https://jolokia.org/) is a Java-based agent that attaches to a JVM and exposes JMX metrics via REST.  Jolokia can be configured to attach to a Java process at startup via JVM opts, or it can be attached to an already running JVM.  Once the Jolokia agent has been attached, JMX metrics in the running application are exposed via a REST interface that returns the metrics in a JSON document.  Once exposed, you can use a tool like [Metricbeat](https://www.elastic.co/guide/en/beats/metricbeat/current/index.html) to pull and aggregate metrics.

## Attaching Jolokia Agent via JVM Opts

Reference: [https://jolokia.org/reference/html/agents.html\#agents-jvm](https://jolokia.org/reference/html/agents.html#agents-jvm)

Add the following to your process's JVM opts:

`-javaagent:/data/software/jolokia/jolokia-jvm-1.3.6-agent.jar=port=12301,host=0.0.0.0`

and replace the path to the jolokia agent jar \(available [here](https://jolokia.org/download.html)\) and the port.

## Attaching Jolokia to a Running JVM

​​​The first step is to locate the process ID of the Java process you'd like to monitor:

`ps aux | grep -i <name of process>`

The second column represents the PID of the running JVM.  The following command will attach the agent to the running process \(**Note: you must run the agent as the same user running the Java process\)**:

`java -jar jolokia-jvm-1.3.6-agent.jar --host 0.0.0.0 --port <desired port> start <process PID>`

For some reason I've noticed this command appears to fail based on the output.  In order to verify it worked, run the following:

`java -jar jolokia-jvm-1.3.6-agent.jar --host 0.0.0.0 --port <port> status <process PID>`

The following command should also show you a process listening for TCP connections on 0.0.0.0:&lt;port&gt;:

`sudo netstat -lnptu | grep <port>`

Once you confirm the agent has attached to the process, you should be able to navigate to [http://&lt;process\_host&gt;:&lt;port&gt;/jolokia/](http://<process_host>:<port>/jolokia/) in a web browser and get a small JSON document in response.  You should also be able to request the following URL in order to see all available MBeans:

`http://<process_host>:<port>/jolokia/search/*:*/`

This request will return a JSON document like the following \(this output is specifically for the HBase REST Service; your process will display different metrics\):

`{`

`"request":{`

```
  "mbean":"\*:\*",

  "type":"search"
```

`},`

`"value":[`

```
  "jolokia:type=Config",

  "JMImplementation:type=MBeanServerDelegate",

  "java.lang:type=Runtime",

  "Hadoop:name=MetricsSystem,service=HBase,sub=Control",

  "java.lang:type=Threading",

  "java.lang:type=OperatingSystem",

  "Hadoop:name=MetricsSystem,service=HBase,sub=Stats",

  "java.lang:name=Code Cache,type=MemoryPool",

  "java.nio:name=direct,type=BufferPool",

  "java.lang:type=Compilation",

  "java.lang:name=CodeCacheManager,type=MemoryManager",

  "java.util.logging:type=Logging",

  "java.lang:type=ClassLoading",

  "java.lang:name=Metaspace Manager,type=MemoryManager",

  "jolokia:type=Discovery",

  "java.lang:name=Metaspace,type=MemoryPool",

  "java.lang:name=Par Eden Space,type=MemoryPool",

  "java.lang:name=ParNew,type=GarbageCollector",

  "jmx4perl:type=Config",

  "java.lang:name=ConcurrentMarkSweep,type=GarbageCollector",

  "Hadoop:name=JvmMetrics,service=HBase",

  "jolokia:type=ServerHandler",

  "java.lang:name=Compressed Class Space,type=MemoryPool",

  "java.lang:type=Memory",

  "java.nio:name=mapped,type=BufferPool",

  "java.lang:name=Par Survivor Space,type=MemoryPool",

  "Hadoop:name=REST,service=HBase",

  "com.sun.management:type=DiagnosticCommand",

  "Hadoop:name=UgiMetrics,service=HBase",

  "com.sun.management:type=HotSpotDiagnostic",

  "java.lang:name=CMS Old Gen,type=MemoryPool"
```

`],`

`"timestamp":1495570152,`

`"status":200`

`}`

The "value" field contains a list of MBean keys, which themselves each have a list of metric values.  These values can also be accessed via REST.  For example, to see all metrics available for the MBean "java.lang:type=Memory​", perform a GET on the following URL:

[http://&lt;process\_host&gt;:&lt;port&gt;/jolokia/read/​java.lang:type=Memory/\*](http://<process_host>:<port>/jolokia/read/​java.lang:type=Memory/*)

Which returns:

{

"request": {

```
"mbean": "java.lang:type=Memory",

"type": "read"
```

},

"value": {

```
"ObjectPendingFinalizationCount": 0,

"Verbose": true,

"HeapMemoryUsage": {

  "init":[2147483648](tel:%28214%29%20748-3648),

  "committed":[2075918336](tel:%28207%29%20591-8336),

  "max":[2075918336](tel:%28207%29%20591-8336),

  "used": 1559627440

},

"NonHeapMemoryUsage": {

  "init": 2555904,

  "committed": 101498880,

  "max": -1,

  "used": 99462272

},

"ObjectName": {

  "objectName": "java.lang:type=Memory"

}
```

},

"timestamp": 1495658723,

"status": 200

}

All the metrics inside the "value" field are fair game, including the nested ones.  The next step is to configure Metricbeat to pull specific metrics from your service through Jolokia.  Metricbeat is configured to dynamically load modules from a monitored config directory.  You only need to configure your specific module \(as a YAML file\) and then place it into the config directory.  For instance, here's a module that pulls heap and garbage collection metrics from WebHBase:

* module: jolokia

  metricsets: \["jmx"\]

  enabled: true

  period: 10s

  namespace: "jolokia\_metrics"

  hosts: \["[hdpr03mn01.mayo.edu:12300](http://hdpr03mn01.mayo.edu:12300/)","[hdpr03mn02.mayo.edu:12300](http://hdpr03mn02.mayo.edu:12300/)"\]

  path: "/jolokia/"

  jmx.mappings:

  * mbean: 'java.lang:name=ConcurrentMarkSweep,type=GarbageCollector'

    attributes:

    * attr: CollectionTime

      field: gc.cms\_collection\_time

* module: jolokia

  * attr: CollectionCount

    field: gc.cms\_collection\_count

    * mbean: 'java.lang:name=ParNew,type=GarbageCollector'

    attributes:

  * attr: CollectionTime

    field: gc.parnew\_collection\_time

  * attr: CollectionCount

    field: gc.parnew\_collection\_count

    * mbean: 'java.lang:type=Memory'

    attributes:

  * attr: HeapMemoryUsage

    field: memory.heap\_usage

  * attr: NonHeapMemoryUsage

    field: memory.non\_heap\_usage

Here's some more information on the Jolokia module: [https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-jolokia.html](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-jolokia.html)

The next page shows a little more of how fields are mapped: [https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-jolokia-jmx.html](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-jolokia-jmx.html)

The first 5 lines of this module shouldn't ever change.  The hosts field is an array of hostnames \*with\* the port included.  The path field should also stay the same.  The JMX mappings are where you're able to customize which metrics you include.  The mbean field would be taked from the list of MBeans retrieved from the initial query '/jolokia/search/\*:\*/' \(such as 'java.lang:type=Memory'\).  The attr field is the title of the metric \(which, in the case of 'java.lang:type=Memory', you would retrieve from the response of '/jolokia/read/java.lang:type=Memory/\*'\).  The next line 'field' is what the field gets mapped to in Elasticsearch.

After saving and dropping the YAML file into the config directory, Metricbeat should load it automatically and begin pushing data to Elasticsearch, where you can then build graphs on the underlying data in Kibana or Grafana.  In Grafana, for instance, you can add a graph, specify datasource "elasticsearch-metricbeat\*", and your metrics will be avaiable as such \(for instance, the CMS CollectionTime attribute specified above\):

jolokia.jolokia\_metrics.gc.cms\_collection\_time

## ​**Addendum - Attaching Agent at Process Start**

It's not always convenient to attach the agent to the Java process on the fly, since restarting the process will likely change the PID and complicate the matter.  In order to deploy a Java process with the agent automatically attached, you can add the following JVM flag to your "java -jar" command:

-javaagent:/data/software/jolokia/jolokia-jvm-1.3.6-agent.jar=port=12301,host=0.0.0.0

The syntax is odd, but it must be in exactly this format: "-javaagent:&lt;path\_to\_agent\_jar&gt;=key1=val1,key2=val2,..."

