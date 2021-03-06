---
layout: global
title: Launching Spark on YARN
---

Support for running on [YARN (Hadoop
NextGen)](http://hadoop.apache.org/docs/r2.0.2-alpha/hadoop-yarn/hadoop-yarn-site/YARN.html)
was added to Spark in version 0.6.0, and improved in subsequent releases.

# Preparations

Running Spark-on-YARN requires a binary distribution of Spark which is built with YARN support.
Binary distributions can be downloaded from the Spark project website. 
To build Spark yourself, refer to the [building with maven guide](building-with-maven.html).

# Configuration

Most of the configs are the same for Spark on YARN as for other deployment modes. See the Configuration page for more information on those.  These are configs that are specific to Spark on YARN.

Environment variables:

* `SPARK_YARN_USER_ENV`, to add environment variables to the Spark processes launched on YARN. This can be a comma separated list of environment variables, e.g. `SPARK_YARN_USER_ENV="JAVA_HOME=/jdk64,FOO=bar"`.

System Properties:

* `spark.yarn.applicationMaster.waitTries`, property to set the number of times the ApplicationMaster waits for the the spark master and then also the number of tries it waits for the Spark Context to be intialized. Default is 10.
* `spark.yarn.submit.file.replication`, the HDFS replication level for the files uploaded into HDFS for the application. These include things like the spark jar, the app jar, and any distributed cache files/archives.
* `spark.yarn.preserve.staging.files`, set to true to preserve the staged files(spark jar, app jar, distributed cache files) at the end of the job rather then delete them.
* `spark.yarn.scheduler.heartbeat.interval-ms`, the interval in ms in which the Spark application master heartbeats into the YARN ResourceManager. Default is 5 seconds. 
* `spark.yarn.max.executor.failures`, the maximum number of executor failures before failing the application. Default is the number of executors requested times 2 with minimum of 3.
* `spark.yarn.historyServer.address`, the address of the Spark history server (i.e. host.com:18080). The address should not contain a scheme (http://). Defaults to not being set since the history server is an optional service. This address is given to the Yarn ResourceManager when the Spark application finishes to link the application from the ResourceManager UI to the Spark history server UI. 

By default, Spark on YARN will use a Spark jar installed locally, but the Spark jar can also be in a world-readable location on HDFS. This allows YARN to cache it on nodes so that it doesn't need to be distributed each time an application runs. To point to a jar on HDFS, export SPARK_JAR=hdfs:///some/path.

# Launching Spark on YARN

Ensure that HADOOP_CONF_DIR or YARN_CONF_DIR points to the directory which contains the (client side) configuration files for the Hadoop cluster.
These configs are used to write to the dfs and connect to the YARN ResourceManager.

There are two deploy modes that can be used to launch Spark applications on YARN. In yarn-cluster mode, the Spark driver runs inside an application master process which is managed by YARN on the cluster, and the client can go away after initiating the application. In yarn-client mode, the driver runs in the client process, and the application master is only used for requesting resources from YARN.

Unlike in Spark standalone and Mesos mode, in which the master's address is specified in the "master" parameter, in YARN mode the ResourceManager's address is picked up from the Hadoop configuration.  Thus, the master parameter is simply "yarn-client" or "yarn-cluster".

To launch a Spark application in yarn-cluster mode:

    ./bin/spark-submit --class path.to.your.Class --master yarn-cluster [options] <app jar> [app options]
    
For example:

    $ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
        --master yarn-cluster \
        --num-executors 3 \
        --driver-memory 4g \
        --executor-memory 2g \
        --executor-cores 1
        examples/target/scala-{{site.SCALA_BINARY_VERSION}}/spark-examples-assembly-{{site.SPARK_VERSION}}.jar \
        yarn-cluster 5

The above starts a YARN client program which starts the default Application Master. Then SparkPi will be run as a child thread of Application Master. The client will periodically poll the Application Master for status updates and display them in the console. The client will exit once your application has finished running.  Refer to the "Viewing Logs" section below for how to see driver and executor logs.

To launch a Spark application in yarn-client mode, do the same, but replace "yarn-cluster" with "yarn-client".  To run spark-shell:

    $ MASTER=yarn-client ./bin/spark-shell

## Adding additional jars

In yarn-cluster mode, the driver runs on a different machine than the client, so SparkContext.addJar won't work out of the box with files that are local to the client. To make files on the client available to SparkContext.addJar, include them with the `--jars` option in the launch command. 

    $ ./bin/spark-submit --class my.main.Class \
        --master yarn-cluster \
        --jars my-other-jar.jar,my-other-other-jar.jar
        my-main-jar.jar
        yarn-cluster 5

# Viewing logs

In YARN terminology, executors and application masters run inside "containers". YARN has two modes for handling container logs after an application has completed. If log aggregation is turned on (with the yarn.log-aggregation-enable config), container logs are copied to HDFS and deleted on the local machine. These logs can be viewed from anywhere on the cluster with the "yarn logs" command.

    yarn logs -applicationId <app ID>
    
will print out the contents of all log files from all containers from the given application.

When log aggregation isn't turned on, logs are retained locally on each machine under YARN_APP_LOGS_DIR, which is usually configured to /tmp/logs or $HADOOP_HOME/logs/userlogs depending on the Hadoop version and installation. Viewing logs for a container requires going to the host that contains them and looking in this directory.  Subdirectories organize log files by application ID and container ID.

# Important notes

- Before Hadoop 2.2, YARN does not support cores in container resource requests. Thus, when running against an earlier version, the numbers of cores given via command line arguments cannot be passed to YARN.  Whether core requests are honored in scheduling decisions depends on which scheduler is in use and how it is configured.
- The local directories used by Spark executors will be the local directories configured for YARN (Hadoop YARN config yarn.nodemanager.local-dirs). If the user specifies spark.local.dir, it will be ignored.
- The --files and --archives options support specifying file names with the # similar to Hadoop. For example you can specify: --files localtest.txt#appSees.txt and this will upload the file you have locally named localtest.txt into HDFS but this will be linked to by the name appSees.txt, and your application should use the name as appSees.txt to reference it when running on YARN.
- The --jars option allows the SparkContext.addJar function to work if you are using it with local files and running in yarn-cluster mode. It does not need to be used if you are using it with HDFS, HTTP, HTTPS, or FTP files.
