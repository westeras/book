# Pulling Jolokia JMX Metrics with Metricbeat

[Metricbeat](https://www.elastic.co/guide/en/beats/metricbeat/current/index.html) is an Elastic beat module that allows you to pull metric data from a variety of different sources and publish it to Elasticsearch for aggregation and querying.  This guide will describe how to pull metrics from Jolokia, the setup of which was described in a previous [chapter](/java/jolokia-jmx-reporting.md).

Metricbeat can be configured to dynamically load modules from a monitored config directory.  This allows you to configure a specific module \(as a YAML file\) and then place it into the config directory.  Metricbeat will automatically detect and incorporate the new module into its metrics querying process.  Here's an example of a config module for a Jolokia metrics endpoint:

```yaml
- module: jolokia
  metricsets: ["jmx"]
  enabled: true
  period: 10s
  namespace: "jolokia_metrics"
  hosts: ["hdpr03mn01.mayo.edu:12300","hdpr03mn02.mayo.edu:12300"]
  path: "/jolokia/"
  jmx.mappings:
    - mbean: 'java.lang:name=ConcurrentMarkSweep,type=GarbageCollector'
      attributes:
        - attr: CollectionTime
          field: gc.cms_collection_time
- module: jolokia
        - attr: CollectionCount
          field: gc.cms_collection_count
    - mbean: 'java.lang:name=ParNew,type=GarbageCollector'
      attributes:
        - attr: CollectionTime
          field: gc.parnew_collection_time
        - attr: CollectionCount
          field: gc.parnew_collection_count
    - mbean: 'java.lang:type=Memory'
      attributes:
        - attr: HeapMemoryUsage
          field: memory.heap_usage
        - attr: NonHeapMemoryUsage
          field: memory.non_heap_usage
```

Here's some more information on the Jolokia module: [https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-jolokia.html](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-module-jolokia.html)

The next page shows a little more of how fields are mapped: [https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-jolokia-jmx.html](https://www.elastic.co/guide/en/beats/metricbeat/current/metricbeat-metricset-jolokia-jmx.html)

The first 5 lines of this module shouldn't ever change.  The hosts field is an array of hostnames \*with\* the port included.  The path field should also stay the same.  The JMX mappings are where you're able to customize which metrics you include.  The mbean field would be taken from the list of MBeans retrieved from the initial query '/jolokia/search/\*:\*/' \(such as 'java.lang:type=Memory'\).  The attr field is the title of the metric \(which, in the case of 'java.lang:type=Memory', you would retrieve from the response of '/jolokia/read/java.lang:type=Memory/\*'\).  The next line 'field' is what the field gets mapped to in Elasticsearch.

After saving and dropping the YAML file into the config directory, Metricbeat should load it automatically and begin pushing data to Elasticsearch, where you can then build graphs on the underlying data in Kibana or Grafana.  In Grafana, for instance, you can add a graph, specify datasource "elasticsearch-metricbeat\*", and your metrics will be available as such \(for instance, the CMS CollectionTime attribute specified in the YAML module above\):

`jolokia.jolokia_metrics.gc.cms_collection_time`

