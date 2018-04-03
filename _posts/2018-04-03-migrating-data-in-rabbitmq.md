---
layout: post
title:  "Migrating data in RabbitMQ"
date:   2018-04-03 16:41:00 -0300
lang: en
categories: rabbitmq amqp message queue docker go golang
description: How to migrate RabbitMQ using Shovel plugin
---
# Migrating data in RabbitMQ

While working on a project, we had the need to migrate our RabbitMQ server, as well to upgrade the version. In this tutorial, I want to give details on how to migrate a RabbitMQ setup to a new installation, including all exchanges, queues and the messages already stored using the **shovel** plugin.

## Setting-up our workbench
To demonstrate this tutorial, I will use docker-compose. There will be two 2 containers, each running a different version of RabbitMQ. The **old** container is running RabbitMQ 3.4, while the **new** container is running RabbitMQ 3.7, which happens to be the latest version available.

All files used in this tutorial is available in [this GitHub repository](https://github.com/gmmoreira).

First, this is the snippet for the `docker-compose.yml`.
```yaml
version: "3"

services:
  rabbitmq_old:
    build:
      context: .
      args:
        rabbitmq_version: 3.4
    ports:
      - "15678:15672"
      - "5678:5672"
  rabbitmq_new:
    build:
      context: .
      args:
        rabbitmq_version: 3.7
    ports:
      - "15679:15672"
      - "5679:5672"
```

The **rabbitmq_old** have its management interface port bind to 15678 in the host and its AMQP protocol port to 5678. **rabbitmq_new** have its management interface and AMQP protocol to 15679 and 5679 respectively.

The Dockerfile is pretty simple:
```Dockerfile
ARG rabbitmq_version
FROM rabbitmq:$rabbitmq_version

RUN rabbitmq-plugins enable rabbitmq_management && \
  rabbitmq-plugins enable rabbitmq_shovel_management && \
  rabbitmq-plugins enable rabbitmq_shovel
```
There's an ARG `rabbitmq_version` variable, so we can specify the RabbitMQ version from the docker-compose file and have a single Dockerfile. We also run a few commands to enable the management plugin, as well the shovel and management shovel plugins.

It's not in the context of this tutorial to explain how Docker and docker-compose works. To start everything, run on a terminal:
```bash
docker-compose up
```
We now have two instances of RabbitMQ running.

To populate just a little our servers, I created a couple of Go programs.

publisher.go
```go
package main

import (
    "fmt"
    "github.com/streadway/amqp"
    "log"
    "os"
)

func failError(err error, message string) {
    formattedMessage := fmt.Sprintf("%s - %s", message, err)
    log.Panic(formattedMessage)
    panic(formattedMessage)
}

func connect() *amqp.Connection {
    rabbitmq_url := os.Getenv("RABBITMQ_URL")
    if rabbitmq_url == "" {
        rabbitmq_url = "amqp://guest:guest@localhost:5672/"
    }
    conn, err := amqp.Dial(rabbitmq_url)
    if err != nil {
        failError(err, "Failed to connect to RabbitMQ")
    }

    return conn
}

func createChannel(conn *amqp.Connection) *amqp.Channel {
    ch, err := conn.Channel()
    if err != nil {
        failError(err, "Failed to open channel on RabbitMQ")
    }

    return ch
}

func declareExchange(ch *amqp.Channel) {
    err := ch.ExchangeDeclare("tutorial", "topic", true, false, false, false, nil)
    if err != nil {
        failError(err, "Failed to declare exchange on RabbitMQ")
    }
}

func main() {
    conn := connect()
    defer conn.Close()

    ch := createChannel(conn)
    defer ch.Close()

    declareExchange(ch)

    msg := amqp.Publishing{
        ContentType:  "plain/text",
        DeliveryMode: amqp.Persistent,
        Body:         []byte("Message sample"),
    }

    log.Print("Publishing message in tutorial exchage, routing key tutorial.key")
    ch.Publish("tutorial", "tutorial.key", false, false, msg)
}
```
consumer.go
```go
package main

import (
    "fmt"
    "github.com/streadway/amqp"
    "log"
    "os"
)

func failError(err error, message string) {
    formattedMessage := fmt.Sprintf("%s - %s", message, err)
    log.Panic(formattedMessage)
    panic(formattedMessage)
}

func connect() *amqp.Connection {
    rabbitmq_url := os.Getenv("RABBITMQ_URL")
    if rabbitmq_url == "" {
        rabbitmq_url = "amqp://guest:guest@localhost:5672/"
    }
    conn, err := amqp.Dial(rabbitmq_url)
    if err != nil {
        failError(err, "Failed to connect to RabbitMQ")
    }

    return conn
}

func createChannel(conn *amqp.Connection) *amqp.Channel {
    ch, err := conn.Channel()
    if err != nil {
        failError(err, "Failed to open channel on RabbitMQ")
    }

    return ch
}

func declareQueue(ch *amqp.Channel) amqp.Queue {
    queue, err := ch.QueueDeclare("tutorial.queue", true, false, false, false, nil)
    if err != nil {
        failError(err, "Failed to declare queue on RabbitMQ")
    }

    return queue
}

func declareBinding(ch *amqp.Channel) {
    err := ch.QueueBind("tutorial.queue", "tutorial.key", "tutorial", false, nil)
    if err != nil {
        failError(err, "Failed to declare queue binding on RabbitMQ")
    }
}

func main() {
    conn := connect()
    defer conn.Close()

    ch := createChannel(conn)
    defer ch.Close()

    declareQueue(ch)
    declareBinding(ch)

    consumer, err := ch.Consume("tutorial.queue", "queue.consumer", false, false, false, false, nil)

    if err != nil {
        failError(err, "Failed to consume queue")
    }

    forever := make(chan boo
l)
    go func() {
        for msg := range consumer {
            defer msg.Ack(false)
            log.Printf("Received message: %s", msg.Body)
        }
    }()

    <-forever
}
```
They are pretty simple. `publisher` declares a **tutorial** exchange, of type `topic`. It then publishes a `Sample message` body, with routing key **tutorial.key**.

`consumer` declares a queue **tutorial.queue** and a binding to **tutorial** exchange with routing key **tutorial.key**. It then consumes all messages in the queue and finishes.

By default, both scripts will connect to AMQP localhost:5672, but we can override this with an environment variable `RABBITMQ_URL`.

Let's run both publisher and consumer once to setup everything on our old server.

```bash
RABBITMQ_URL=amqp://localhost:5678 ./publisher
RABBITMQ_URL=amqp://localhost:5678 ./consumer
```
If we access the management, we should see the exchange and queue declared.

## Exporting the definitions
The first step is to export the definitions of our old RabbitMQ server. This can be easily made in the management interface, on Overview tab.
By downloading the definitions, we are getting all virtual hosts, users, exchanges, queues and bindings from our broker.

## Importing the definitions
We can know import the definitions in the new broker. Just like exporting, the importing section is also in the Overview tab. Just upload the file and we are done, our new broker should have all definitions from the old broker.

## Migrating messages using Shovel plugin
The Shovel plugin should be available for nearly all RabbitMQ installations as it's a default plugin. The shovel plugin can be configured both statically and dynamically. The static configuration is done in rabbitmq config file, an option which I'm not too fond.

The dynamic configuration can be made through the `rabbitmqctl` program, the management HTTP API or, even easier, the management Web UI. In this tutorial, I will use the Web UI.

The shovel plugin must be enabled in at least one broker. For a migration, I believe the best option is to enable it on the old server so the new server does not need to know anything about the shovel. In my example, I'm enabling the shovel in both brokers.

### Configuring the shovel
First, let's publish a few messages:
```bash
for i in {1..100}; do RABBITMQ_URL=amqp://localhost:5678 ./publisher; done
```
This will publish 100 messages, which will be stored on **tutorial.queue**.

Let's know configure the shovel for this queue. In the management Web UI of the old broker, go to the Admin tab and click in the Shovel Management menu entry. Expand the **Add a new shovel** section and let's fill the form.

`Name` can be anything, let's fill **tutorial.queue migration**.
`Source URI` must be filled with the old broker connection URL. As I'm using docker-compose, it's hostname inside the docker network is **rabbitmq_old**.
`amqp://guest:guest@rabbitmq_old:5672`
In the drop-down, let's pick `Queue` and fill the name of our queue **tutorial.queue**.
Just like `Source URI`, `Destination URI` must be filled with the new broker connection. In our case, `amqp://guest:guest@rabbitmq_new:5672`.
In the drop-down, let's pick `Queue` too and fill the same queue *tutorial.queue*.
We can leave all other fields empty. If you want to know more about them, the Web UI provides explanations.
To finish, click `Add shovel` button.

We can verify the status of the configured shovels in the Shovel Status menu entry.

Now that the shovel has been configured, if we check the queue in the old broker it should be empty and the queue in the new broker must have the 100 messages we published earlier.

Let's publish a few more messages:
```bash
for i in {1..100}; do RABBITMQ_URL=amqp://localhost:5678 ./publisher; done
```

The queue in the old broker should still be empty and the queue in the new broker will contain another 100 messages. When we configured the shovel with a queue in both source and destination, the shovel will automatically forward messages already stored the source queue, as well the new messages that get published for that queue.
