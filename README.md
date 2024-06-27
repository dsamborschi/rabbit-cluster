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

We’ll start by creating a docker-compose.yml file to define our RabbitMQ setup. Below is a sample configuration that includes a 3 node RabbitMQ cluster and Haproxy load balancer:

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
        ports:
            - 15674:15672
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

version: '3': This specifies the version of the Docker Compose file format that we are using.
services: This section defines the RabbitMQ service that we want to deploy. In this case, we are using the official RabbitMQ Docker image with the management plugin enabled.
image: rabbitmq:management: This specifies the RabbitMQ Docker image we want to use.
container_name: rabbitmq: This assigns a name to the RabbitMQ container.
environment: This section sets environment variables for the RabbitMQ container. In this example, we are setting the default username and password to "guest". Note that this is not recommended for production environments.
ports: This section maps the ports used by RabbitMQ to the corresponding ports on the host machine. In this case, we are mapping the port 5672 for AMQP communication and the port 15672 for the RabbitMQ management interface.
networks: This section specifies the network settings for the RabbitMQ container. In this example, we are using the default Docker bridge network.

## Step 4: Verifying Installation:

Once the containers are up and running, you can verify the installation by accessing the RabbitMQ Management UI in your web browser. Navigate to http://localhost:15672 view the Management dashboard. From here, you can monitor RabbitMQ connections, channels, exchanges, queues, and streams in real time.

