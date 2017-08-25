# Persisting RDDs to Disk Between Runs

For a long running Spark job, debugging gets annoying when you have to wait several minutes for the job to get to the part in the process you're trying to debug.  One way around this is to save an RDD to hard disk at an intermediate step in the process, allowing you to load it directly and avoid all the initial processing.

