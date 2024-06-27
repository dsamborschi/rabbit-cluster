# How to set up a simple dockerized RabbitMQ cluster

RabbitMQ is the most widely deployed open source message broker.

RabbitMQ is lightweight and easy to deploy on premises and in the cloud. It supports multiple messaging protocols. RabbitMQ can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.

RabbitMQ runs on many operating systems and cloud environments, and provides a wide range of developer tools for most popular languages.

In my current workplace we use a 3 nodes cluster running on 3 separate Linux VM machines. We do have a dev environment but sometimes there is a need to span something quick and dirty.  This step-by-step guide will walk you through installing RabbitMQ cluster using Docker Compose and Docker Desktop.


In a nutshel, this is what we are going to create. Load balancing is optional and to be honest not recommended option for production RabbitMQ cluster. 

![image](https://github.com/dsamborschi/Rabbit-Cluster/assets/3628896/444069b3-a48d-4604-99a9-f542ffc00ff8)


## Step 1: Setting up Docker Compose and Docker Desktop:

Docker Compose is a tool for defining and running multi-container Docker applications. It allows you to manage complex setups with ease. 

Leveraging Docker Compose for RabbitMQ deployment streamlines the process abstracting away concerns regarding underlying infrastructure.  If you dont have these tools installed, simply follow the links below to istall.

https://docs.docker.com/desktop/install/windows-install/

https://docs.docker.com/compose/

## Step 2: Writing the Docker Compose File:

We’ll start by creating a docker-compose.yml file to define our RabbitMQ setup. Below is a sample configuration that includes a 3 node RabbitMQ cluster and HAproxy load balancer:

```yaml
version: "3.8"

name: rabbitmq-cluster
services:
    rabbitmq1:
        build: rabbitmq/.
        container_name: rabbitmq1
        volumes:
            - "./data1:/var/lib/rabbitmq/mnesia/"
            - ./rabbitmq/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie
            - ./rabbitmq/cluster-entrypoint.sh:/usr/local/bin/cluster-entrypoint.sh
        hostname: "rabbitmq1"
        environment:
            - RABBITMQ_ERLANG_COOKIE='12345'
            - RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq-custom
        entrypoint: /usr/local/bin/cluster-entrypoint.sh
        networks: 
            - rabbitmq-cluster-network
    rabbitmq2:
        build: rabbitmq/.
        container_name: rabbitmq2
        environment:
            - JOIN_CLUSTER_HOST=rabbitmq1
            - RABBITMQ_ERLANG_COOKIE='12345'
            - RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq-custom
        depends_on:
          - rabbitmq1
        volumes:
            - "./data2:/var/lib/rabbitmq/mnesia/"
            - ./rabbitmq/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie
            - ./rabbitmq/cluster-entrypoint.sh:/usr/local/bin/cluster-entrypoint.sh
        hostname: "rabbitmq2"
        entrypoint: /usr/local/bin/cluster-entrypoint.sh
        networks: 
            - rabbitmq-cluster-network
    rabbitmq3:
        build: rabbitmq/.
        container_name: rabbitmq3
        depends_on:
          - rabbitmq1
          - rabbitmq2
        environment:
          - JOIN_CLUSTER_HOST=rabbitmq1
          - RABBITMQ_ERLANG_COOKIE='12345'
          - RABBITMQ_CONFIG_FILE=/etc/rabbitmq/rabbitmq-custom
        volumes:
          - "./data3:/var/lib/rabbitmq/mnesia/"
          - ./rabbitmq/.erlang.cookie:/var/lib/rabbitmq/.erlang.cookie
          - ./rabbitmq/cluster-entrypoint.sh:/usr/local/bin/cluster-entrypoint.sh
        hostname: "rabbitmq3"
        entrypoint: /usr/local/bin/cluster-entrypoint.sh
        networks: 
            - cluster-network
    haproxy:
        image: haproxy:1.9
        container_name: haproxy
        volumes:
          - ./haproxy:/usr/local/etc/haproxy/
        depends_on:
          - rabbitmq1
          - rabbitmq2
          - rabbitmq3
        ports:
          - 15672:15672
          - 5672:5672
        networks: 
            - rabbitmq-cluster-network
networks:
    rabbitmq-cluster-network:
     driver: bridge
```

## In this configuration:

- version: '3.8': This specifies the version of the Docker Compose file format that we are using.
- services: This section defines the RabbitMQ service that we want to deploy. In this case, we are using the official RabbitMQ Docker image with the management plugin enabled.
- container_name: rabbitmq: This assigns a name to the RabbitMQ container.
- environment: This section sets environment variables for the RabbitMQ container. In this example, we are setting the default username and password to "guest". Note that this is not recommended for production environments.
- ports: This section maps the ports used by RabbitMQ to the corresponding ports on the host machine. In this case, we are mapping the port 5672 for AMQP communication and the port 15672 for the RabbitMQ management interface. For simplicity, HAproxy was used to balance the incoming traffic.
- networks: This section specifies the network settings for the RabbitMQ container. In this example, we are using the rabbitmq-cluster-network Docker bridge network.

## Step 3: Writing the RabitMQ Docker file:

```yaml
FROM rabbitmq:3-management-alpine
COPY rabbitmq-custom.conf /etc/rabbitmq/rabbitmq-custom.conf
COPY definitions-custom.json /etc/rabbitmq/definitions-custom.json
```

## Step 4: Writing the RabitMQ custom configuration:

```yaml
# default settings
loopback_users.guest = true
listeners.tcp.default = 5672
hipe_compile = false
management.listener.port = 15672
management.listener.ssl = false

# Load the queues we initially want
management.load_definitions = /etc/rabbitmq/definitions-custom.json
vm_memory_high_watermark.relative = 0.8
```

## Step 5: Writing the RabitMQ custom definitions (optional):

You can save some time to tell RabbbitMQ nodes how you want your users and nodes to be configured. To do so we can use the custom definitions file below. It is an optional step, but very helfull if you want to quickly set up/clone a new testing environment. 

## Step 6: Writing the RabitMQ cluster entrypoint bash script:

```yaml
#!/bin/bash

set -e

# Change .erlang.cookie permission
chmod 400 /var/lib/rabbitmq/.erlang.cookie

# Get hostname from enviromant variable
HOSTNAME=`env hostname`
echo "Starting RabbitMQ Server For host: " $HOSTNAME

if [ -z "$JOIN_CLUSTER_HOST" ]; then
    /usr/local/bin/docker-entrypoint.sh rabbitmq-server &
    sleep 5
    rabbitmqctl wait /var/lib/rabbitmq/mnesia/rabbit\@$HOSTNAME.pid
else
    /usr/local/bin/docker-entrypoint.sh rabbitmq-server -detached
    sleep 5
    rabbitmqctl wait /var/lib/rabbitmq/mnesia/rabbit\@$HOSTNAME.pid
    rabbitmqctl stop_app
    rabbitmqctl join_cluster rabbit@$JOIN_CLUSTER_HOST
    rabbitmqctl start_app
fi

# Keep foreground process active ...
tail -f /dev/null
```
## Step 7: Verify Docker Desktop containers:

![image](https://github.com/dsamborschi/rabbit-cluster/assets/3628896/0e409440-1b04-46a6-931f-c9a332ccae4d)


## Step 8: Verify installation:

Once the containers are up and running, you can verify the installation by accessing the RabbitMQ Management UI in your web browser. Navigate to http://localhost:15672 view the Management dashboard. From here, you can monitor RabbitMQ cluster, connections, channels, exchanges, queues, and streams in real time.

![image](https://github.com/dsamborschi/rabbit-cluster/assets/3628896/09143255-9af5-4991-8b8a-89bc2b5968b4)


