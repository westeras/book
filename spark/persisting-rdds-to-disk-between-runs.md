# Persisting RDDs to Disk Between Runs

For a long running Spark job, debugging gets annoying when you have to wait several minutes for the job to get to the part in the process you're trying to debug.  One way around this is to save an RDD to hard disk at an intermediate step in the process, allowing you to comment out the early processing and load the RDD directly from disk to avoid all the initial processing.

To save an RDD \(Java, type is JavaRDD&lt;InventoryPlan&gt;\):

`inventoryPlanRDD.saveAsObjectFile("/choose/path/for/persisted/rdd");`

To reload the same RDD:

`JavaRDD<InventoryPlan> inventoryPlanRDD = jsc.objectFile("/path/to/persisted/rdd");`

