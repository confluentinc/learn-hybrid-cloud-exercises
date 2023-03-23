# Hands-on: Configuring MirrorMaker 2

In this hands-on, we'll look at configuring MirrorMaker 2 and replicating data from one cluster to another. 

To follow along, you'll need to have Docker installed. You'll also need to clone the GitHub repo for this course. 

The first thing we'll do is start our Docker container. We will be using a Kafka instance that contains two zookeeper instances, two brokers, a connect cluster, and a schema registry instance. As of the writing of this course, the most recent version of Kafka is 3.3 and Confluent Platform 7.3.2. By the time you take this course, there might have been other releases that break or change how things work and function. If you can't figure out why something isn't working please reach out to us using the [Confluent Community Forum or Public Slack](https://www.confluent.io/community/ask-the-community/).

After you have cloned the GitHub repo start the cluster by typing: 

`docker-compose up -d`

The first time this runs it will take a while as it is downloading all the necessary files. Once it is done make sure that everything is running by typing:

`docker-compose ps`

We need some topics and data in the Kafka instance, to create it, enter into broker’s container from a new terminal tab or window and execute the kafka-topics command:

`docker exec -it broker /bin/bash`

Create the topic:

`kafka-topics --create --topic replicate_me --bootstrap-server broker:29092`

Now let's create some messages that we can have replicated:

`while [[ true ]]; do echo "$RANDOM" |  kafka-console-producer --topic replicate_me --bootstrap-server broker:29092; sleep 1; done &`

Next, we'll enter Kafka Connect Docker container to run our commands

`docker exec -it connect /bin/bash`

To create a connector you'll want to take a look at the [official Kafka Connect documentation](https://docs.confluent.io/platform/current/connect/userguide.html). For this example, we'll just use the /connectors endpoint.

Create a new connector using the JSON format. 

`cat > connectors.json`

```
{
  "name":"test_mirror",
  "config": {
    "connector.class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "name":"test_mirror",
    "source.cluster.alias":"source",
    "topics":"replicate_me",
    "source.cluster.bootstrap.servers":"broker:29092",
    "target.cluster.bootstrap.servers":"broker2:29093",
    "producer.override.bootstrap.servers":"broker2:29093",
    "offset-syncs.topic.replication.factor":"1"
  }
}
```

Hit `Enter` and then `Ctrl-C`

Now let's start our mirroring by performing the `POST` command to create our new connector:

`cat connectors.json | curl -X POST -H 'Content-Type: application/json' localhost:8083/connectors --data-binary @-`

You can verify that the new connector has been created:

`curl localhost:8083/connectors`

Open up a new terminal and connect to broker2:

`docker exec -it broker2 /bin/bash`

Let's look at the topics:

`kafka-topics --list --bootstrap-server broker2:29093`

You'll see our new topic source `source.replicate_me`

Finally, let's take a look at the messages that are being replicated:

`kafka-console-consumer --topic source.replicate_me --bootstrap-server broker2:29093`






