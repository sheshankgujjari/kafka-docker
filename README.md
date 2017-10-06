## Images required

* wurstmeister/zookeeper
* wurstmeister/kafka

## Manual Execution

```
docker-compose up -d
```

Config required for a broker

* broker.id -- unique for each broker
* listeners -- IP of Docker container assigned in the bridge network/compose network
* zookeeper.connect -- the Docker bridge network/ compose network IP of the Zookeeper container


### Zookeeper

First, we start the Zookeeper container. This contains the Zookeeper server binary we need to run.
```
docker run -it --rm wurstmeister/zookeeper /bin/bash
```

Now check the IP from within the container, using `hostname -I`. You should see `172.17.0.2` and it should be the same in any case.

Now we start the Zookeeper server.
```
./bin/zkServer.sh start-foreground
```

### One Kafka Broker

For now, we run only one Kafka broker.

Start the container:
```
docker run -it --rm wurstmeister/kafka /bin/bash
```

Check the IP using `hostname-i` . It should be `172.17.0.3`.

The directory of interest is `$KAFKA_HOME`.
```
cd $KAFKA_HOME
```

You now need to edit the required options in `config/server.properties`. Set the following values:

```
vi config/server.properties
```

Edit the below

* broker.id -- leave it to 0; we have only 1 broker
* listeners -- PLAINTEXT://172.17.0.3:9092, the IP of this broker
* zookeeper.connect -- 172.17.0.2:2181, the IP to reach the Zookeeper instance

We can now start the server:
```
./bin/kafka-server-start.sh config/server.properties
```

### One Producer

We are going to start a single producer by opening a new command prompt

We need to start a `wurstmeister/kafka` container as only that has the required scripts.
So, as before:
```
docker run -it --rm wurstmeister/kafka /bin/bash
$ cd $KAFKA_HOME
```

To get a list of existing topics:
```
$KAFKA_HOME/bin/kafka-topics.sh --list --zookeeper 172.17.0.2
```
The list will be empty

Let us create a topic, `test`:
```
$KAFKA_HOME/bin/kafka-topics.sh --zookeeper 172.17.0.2 --partitions 1 --replication-factor 1 --create --topic test
```

Finally the producer:
```
$KAFKA_HOME/bin/kafka-console-producer.sh --broker-list 172.17.0.3:9092 --topic test
```

You can enter stuff into the command line now. Each line is considered a seperate message.

### One Consumer

Just like the producer case, we need to start the container in a new tab
```
docker run -it --rm wurstmeister/kafka /bin/bash
$ cd $KAFKA_HOME
```

Consume from the beginning:
```
$KAFKA_HOME/bin/kafka-console-consumer.sh --zookeeper 172.17.0.2:2181 --topic test --from-beginning
```

You should be able to see the messages from the producer.

## Automation using Compose

By using the `docker-compose scale` command, setup which permit adding more brokers.

Make sure you use the same image (`wurstmeister/kafka`) and try configure it at start time.

We shall do that by using the commands available from bash, namely `sed`, `hostname`, etc.

So, for each config, below is our value setup:

### Editing broker.id in bash
Needs to be an unique integer.

Note that the brokers (containers) are going to be started on a seperate network. Each container can obviously access it's own IP. So the last (least significant) 8 bits of the IP (for simplicity) would be unique to each broker (assuming reasonable IP allocation and limited number of brokers).

```
export BROKER_ID=$(hostname -i | sed 's/\([0-9]*\.\)*\([0-9]*\)/\2/')
sed -i 's/^\(broker\.id=\).*/\1'$BROKER_ID'/' config/server.properties
```

If required, we can use more number of bits to set the broker id. But for our cases, 251 brokers should be more than what you need on a single host.

### Editing listeners in bash
Needs to be the container's IP. Straight-forward substitution.

```
sed -i 's|^#\(listeners=PLAINTEXT://\)\(:9092\)|\1'`hostname -i`'\2|' config/server.properties
```

### Editing zookeeper.connect in bash
Zookeeper IP connection. We link the Kafka broker containers to the zookeeper container, meaning the hostname zookeeper directly resolves to the correct IP.

```
sed -i 's|^\(zookeeper.connect=\)\(localhost\)\(:2181\)|\1zookeeper\3|' config/server.properties
```

* * *

Now that we have the techniques, `bash -c` command does not have access to the ENV variables inside the container.

We need to mount the folder with the script onto the Kafka image containers and running the script before starting the server.


## Known Issues

Do not bring down using `CTRL-C`. Make sure to use `docke-compose stop` and `docker-compose rm`

To produce/consume, if you use the scripts included in the Kafka bin/ folder, you need to run a container on the same network as the brokers, or alternatively directly attach to one of the brokers.
