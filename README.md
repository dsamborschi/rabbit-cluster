# How to set up a simple dockerized RabbitMQ cluster

RabbitMQ is the most widely deployed open source message broker.

RabbitMQ is lightweight and easy to deploy on premises and in the cloud. It supports multiple messaging protocols. RabbitMQ can be deployed in distributed and federated configurations to meet high-scale, high-availability requirements.

RabbitMQ runs on many operating systems and cloud environments, and provides a wide range of developer tools for most popular languages.

We are going to create a 3 nodes cluster running in docker containers using Docker Desktop. 

Leveraging Docker Compose for RabbitMQ deployment streamlines the process abstracting away concerns regarding underlying infrastructure. This step-by-step guide will walk you through installing RabbitMQ cluster using Docker Compose and Docker Desktop. If you dont have it installed, please see the docker docs or simply follow the link below.

https://docs.docker.com/desktop/install/windows-install/


In a nutshel, this is what we are going to create. Load balancing is optional and to be honest not recommended option for production RabbitMQ cluster. 

![image](https://github.com/dsamborschi/Rabbit-Cluster/assets/3628896/444069b3-a48d-4604-99a9-f542ffc00ff8)
