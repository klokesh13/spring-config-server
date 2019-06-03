# spring-config-server

Spring Cloud Config provides server-side and client-side support for externalized configuration in a distributed system. With the Config Server, you have a central place to manage external properties for applications across all environments. 

We can fetch the configurations from external properties either by file system or git. This config server uses git repository for storing the configurations.

## Usage

Once we run the application, It will be running on default port 8888 and uri would be "http://toris-config.service.consul:8888"

On the client side, we need to map this config server and configuration file to lookup in git repo by the below properties.

**spring.cloud.config.uri=http://toris-config.service.consul:8888
spring.application.name=reader-config-vi**

## Ways of updating client with latest configurations from git repo 

## 1. Actuator

The client may access any value in the Config Server using the traditional mechanisms **(e.g. @ConfigurationProperties, @Value("${…​}")** or through the Environment abstraction). 

By default, the configuration values are read on the client’s startup, and not again. You can force a bean to refresh its configuration - to pull updated values from the Config Server - by annotating the RestController with the Spring Cloud Config **@RefreshScope** and then by triggering a **refresh** event.

**Core dependencies required:**
    
    spring-boot-starter-actuator
    
If we update any configuration on the git repo and if that needs to get updated in client, we need to run the below curl command on the each client application which is mapped with config server.

**curl -X POST http://serviceHostName:8080/actuator/refresh**


## 2. Spring Cloud Bus with RabbitMq

## 2.1 On Client Side

By using Actuator, we can refresh clients. However, in the cloud environment, we would need to go to every single client and reload configuration by accessing actuator endpoint.
To solve this problem, we can use Spring Cloud Bus.


**Core dependencies required:**
    
    spring-cloud-starter-bus-amqp
    
**Configuration:**
    
    spring.rabbitmq.addresses=amqp://toris-nonprod:sKGY5D3Qq8eouWJ9badr@hefty-moose.rmq.cloudamqp.com/toris-config-dev
    
Now we can run the below curl command in any one of the client and an event will get triggered to every client which was registered with rabbitmq address

**curl -X POST http://serviceHostName:8080/actuator/bus-refresh**

## 2.2 On Server Side

In order to make it completely automatic, we need to apply below changes to the config server.

**Core dependencies required:**
    
    - spring-cloud-config-monitor
    
    - spring-cloud-starter-stream-rabbit

**Configuration:**
    
    spring.rabbitmq.addresses=amqp://toris-nonprod:sKGY5D3Qq8eouWJ9badr@hefty-moose.rmq.cloudamqp.com/toris-config-dev

Once the server gets notified about configuration changes, it will broadcast this as a message to RabbitMQ. The client will listen to messages and update its configuration when configuration change event is transmitted. However, how a server will now about the modification?

We need to configure a GitHub Webhook. Let’s go to GitHub and open our repository holding configuration properties. Now, let’s select Settings and Webhook. Let’s click on Add webhook button.

Payload URL is the URL for our config server **‘/monitor’** endpoint. In our case the URL will be something like **http://serviceHostName:8888/monitor**

We just need to change Content type in the drop-down menu to application/json. Next, please leave Secret empty and click on Add webhook button – after that, we are all set.

**NOTE:** If you do not have public ip address or public domain name set up yet, you can publish (simulate) the webhook event for the Config Server by calling the /monitor endpoint with given request body. Then the Config Server will consider it as a webhook event published by the related repository service provider and continue with the rest of the operations.

**curl -v -X POST "http://toris-config.service.consul:8888/monitor" -H "Content-Type: application/json" -H "X-Event-Key: repo:push" -H "X-Hook-UUID: webhook-uuid" -d '{"push": {"changes": []} }'**
