# Deployment

## Zookeeper

Being a distributed application, Storm also uses a ZooKeeper cluster to coordinate various
processes. All of the states associated with the cluster and the various tasks submitted to Storm
are stored in ZooKeeper.

In the ZooKeeper ensemble, one node in the cluster acts as the leader, while the rest are followers.
If the leader node of the ZooKeeper cluster dies, then an election for the new leader takes places
among the remaining live nodes, and a new leader is elected. All write requests coming from clients
are forwarded to the leader node, while the follower nodes only handle the read requests. Also, we
can't increase the write performance of the ZooKeeper ensemble by increasing the number of nodes
because all write operations go through the leader node.

> It is advised to run an odd number of ZooKeeper nodes, as the ZooKeeper cluster keeps working as
> long as the majority (the number of live nodes is greater than *n/2*, where *n* being the number of
> deployed nodes) of the nodes are running. So if we have a cluster of four ZooKeeper nodes (*3 > 4/2*
> ; only one node can die), then we can handle only one node failure, while if we had five nodes (*3 >
5/2*; two nodes can die) in the cluster, then we can handle two node failures.

## Hello world sample

### Spout (Source) sample

This spout does not connect to an external source to fetch data, but randomly generates the data and
emits a continuous stream of records.

```java
public class SampleSpout extends BaseRichSpout {

  private static final long serialVersionUID = 1L;

  private static final Map<Integer, String> map = new HashMap<Integer, String>();

  static {
    map.put(0, "google");
    map.put(1, "facebook");
    map.put(2, "twitter");
    map.put(3, "youtube");
    map.put(4, "linkedin");
  }

  private SpoutOutputCollector spoutOutputCollector;

  @SuppressWarnings("rawtypes")
  public void open(Map conf, TopologyContext context, SpoutOutputCollector spoutOutputCollector) {
    // Open the spout
    this.spoutOutputCollector = spoutOutputCollector;
  }

  public void nextTuple() {
    // Storm cluster repeatedly calls this method to emit continuous
    // stream of tuples.
    final Random rand = new Random();
    // generate the random number from 0 to 4.
    int randomNumber = rand.nextInt(5);
    spoutOutputCollector.emit(new Values(map.get(randomNumber)));
    try {
      Thread.sleep(5000);
    } catch (Exception e) {
      System.out.println("Failed to sleep the thread");
    }
  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {

    // emit the tuple with field "site"
    declarer.declare(new Fields("site"));
  }
}
```

### Bold (Processor) sample

This bolt will consume the tuples emitted by the SampleSpout spout and will print the value of the
field site on the console.

```java
public class SampleBolt extends BaseBasicBolt {

  private static final long serialVersionUID = 1L;

  public void execute(Tuple input, BasicOutputCollector collector) {
    // fetched the field "site" from input tuple.
    String test = input.getStringByField("site");
    // print the value of field "site" on console.
    System.out.println("######### Name of input site is : " + test);
  }

  public void declareOutputFields(OutputFieldsDeclarer declarer) {
  }
}
```

### Topology (DAG)

Create a main `SampleStormTopology` class within the same package. This class creates an instance of
the spout and bolt along with the classes, and chaines them together using a `TopologyBuilder`
class. This class uses `org.apache.storm.LocalCluster` to simulate the Storm cluster.
The `LocalCluster` mode is used for debugging/testing the topology on a developer machine before
deploying it on the Storm cluster. The following is the implementation of the main class for running
locally:

##### Prepare local

```java
package com.stormadvance.storm_example;

import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.topology.TopologyBuilder;

public class SampleStormTopology {

  @SuppressWarnings("resource")
  public static void main(String[] args) throws Exception {
    // create an instance of TopologyBuilder class
    TopologyBuilder builder = new TopologyBuilder();
    // set the spout class
    builder.setSpout("SampleSpout", new SampleSpout(), 2);
    // set the bolt class
    builder.setBolt("SampleBolt", new SampleBolt(), 4).shuffleGrouping("SampleSpout");
    Config conf = new Config();
    conf.setDebug(true);
    // create an instance of LocalCluster class for
    // executing topology in local mode.
    LocalCluster cluster = new LocalCluster();
    // SampleStormTopology is the name of submitted topology
    cluster.submitTopology("SampleStormTopology", conf, builder.createTopology());
    try {
      Thread.sleep(100000);
    } catch (Exception exception) {
      System.out.println("Thread interrupted exception : " + exception);
    }
    // kill the SampleStormTopology
    cluster.killTopology("SampleStormTopology");
    // shutdown the storm test cluster
    cluster.shutdown();
  }
}
```

##### Prepare for cluster

```java
public class SampleStormClusterTopology {

  public static void main(String[] args) throws AlreadyAliveException, InvalidTopologyException {
    // create an instance of TopologyBuilder class
    TopologyBuilder builder = new TopologyBuilder();
    // set the spout class
    builder.setSpout("SampleSpout", new SampleSpout(), 2);
    // set the bolt class
    builder.setBolt("SampleBolt", new SampleBolt(), 4).shuffleGrouping("SampleSpout");
    Config conf = new Config();
    conf.setNumWorkers(3);
    // This statement submit the topology on remote
    // args[0] = name of topology
    try {
      StormSubmitter.submitTopology(args[0], conf, builder.createTopology());
    } catch (AlreadyAliveException alreadyAliveException) {
      System.out.println(alreadyAliveException);
    } catch (InvalidTopologyException invalidTopologyException) {
      System.out.println(invalidTopologyException);
    } catch (AuthorizationException e) {
      // TODO Auto-generated catch block
      e.printStackTrace();
    }
  }
}
```

##### Execute in local

```sh
# go to storm_example

mvn clean install

mvn compile exec:java -Dexec.classpathScope=compile -Dexec.mainClass=com.stormadvance.storm_example.SampleStormTopology
```

##### Execute in cluster

> The structure of the command to execute in cluster (main node) is
>
> ```sh
> storm jar jarName.jar [TopologyMainClass] [Args]
> ```
> The main function of `TopologyMainClass` is to define the topology and submit it to the Nimbus
> machine. The storm jar part takes care of connecting to the Nimbus machine and uploading the JAR
> part.

In storm nimbus (main node):

```shell
mvn clean install

# Move to main node
docker cp target/storm_example-0.0.1-SNAPSHOT-jar-with-dependencies.jar s02-nimbus:/sample_topology.jar

# Execute in container
docker exec -it s02-nimbus storm jar /sample_topology.jar com.stormadvance.storm_example.SampleStormClusterTopology storm_example
```

### Storm cluster operations

#### Deactivate

Storm supports the deactivating a topology. In the deactivated state, spouts will not emit any new
tuples into the pipeline, but the processing of the already emitted tuples will continue.
Deactivating the topology does not free the Storm resource. If topology is activated, the spout
again starts emitting tuples.

> ```sh
> storm deactivate topologyName
> ```

##### Deactivating sample topology

```sh
docker exec -it s02-nimbus storm deactivate storm_example
```

#### Kill

> ```sh
> storm kill topologyName
> ```

**Storm topologies are never-ending processes**. To stop a topology, we need to kill it. When
killed, the topology first enters into the deactivation state, processes all the tuples already
emitted into it, and then stops. Once it is killed, it will free all the Storm resources allotted to
this topology.

##### Killing sample topology

```sh
docker exec -it s02-nimbus storm deactivate storm_example
```

##### Rebalance

##### Change log level settings

> ```
> storm set_log_level [topology name] -l [logger name]=[LEVEL]:[TIMEOUT]
> ```

The `TIMEOUT` is the time in seconds. The log level will go back to normal after the timeout time.
The value of `TIMEOUT` is mandatory if you are setting the log level to `DEBUG`/`ALL`.

##### Chaning log level settings

```sh
docker exec -it s02-nimbus storm set_log_level storm_example -l ROOT=DEBUG:30
```

### Storm UI

**While submitting a topology to the cluster, the user first needs to make sure that the value of
the Free slots column should not be zero; otherwise, the topology doesn't get any worker for
processing and will wait in the queue until a workers becomes free**.

A node with the status leader is an active master while the node with the status Not a Leader is
a `passive master`.