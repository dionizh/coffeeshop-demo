# Coffeeshop Demo with Quarkus

This directory contains a set of demo around _reactive_ in Quarkus with Kafka.
It demonstrates the elasticity and resilience of the system.

## Build

```bash
mvn clean package
```

## Prerequisites

Install Kafka locally for the Kafka tools e.g.

```bash
brew install kafka
```

Run Kafka with:

```bash
docker-compose up
```

In case of previous run, you can clean the state with

```bash
docker-compose down
docker-compose rm
```

Then, create the `orders` topic with `./create-topics.sh`

# Run the demo

You need to run:

* the coffee shop service
* the HTTP barista
* the Kafka barista

Im 3 terminals: 

```bash
cd coffeeshop-service
mvn compile quarkus:dev
```

```bash
cd barista-http
java -Dbarista.name=tom -jar target/barista-http-1.0-SNAPSHOT-runner.jar
```

```bash
cd barista-kafka
mvn compile quarkus:dev
```

# Execute with HTTP

The first part of the demo shows HTTP interactions:

* Barista code: `me.escoffier.quarkus.coffeeshop.BaristaResource`
* CoffeeShop code: `me.escoffier.quarkus.coffeeshop.CoffeeShopResource.http`
* Generated client: `me.escoffier.quarkus.coffeeshop.http.BaristaService`

Order coffees with:

```bash
while [ true ]
do
http POST :8080/http product=latte name=clement
http POST :8080/http product=expresso name=neo
http POST :8080/http product=mocha name=flore
done
```

Stop the HTTP Barista, you can't order coffee anymore.

# Execute with Kafka

* Barista code: `me.escoffier.quarkus.coffeeshop.KafkaBarista`: Read from `orders`, write to `queue`
* Bridge in the CoffeeShop: `me.escoffier.quarkus.coffeeshop.messaging.CoffeeShopResource#messaging` just enqueue the orders in a single thread (one counter)
* Get prepared beverages on `me.escoffier.quarkus.coffeeshop.dashboard.BoardResource` and send to SSE

* Open browser to http://localhost:8080/
* Order coffee with:

```bash
http POST :8080/messaging product=latte name=clement
http POST :8080/messaging product=expresso name=neo
http POST :8080/messaging product=mocha name=flore
```

# Baristas do breaks

1. Stop the Kafka barista
1. Continue to enqueue order
```bash
http POST :8080/messaging product=frappuccino name=clement
http POST :8080/messaging product=chai name=neo
http POST :8080/messaging product=hot-chocolate name=flore
```
1. On the dashboard, the orders are in the "IN QUEUE" state
1. Restart the barista
1. They are processed

# 2 baristas are better

1. Start a second barista with: 
```bash
java -Dquarkus.http.port=9095 -Dbarista.name=tom -jar target/barista-kafka-1.0-SNAPSHOT-runner.jar
```
1. Order more coffee
```bash
http POST :8080/messaging product=frappuccino name=clement
http POST :8080/messaging product=chai name=neo
http POST :8080/messaging product=hot-chocolate name=flore
http POST :8080/messaging product=latte name=clement
http POST :8080/messaging product=expresso name=neo
http POST :8080/messaging product=mocha name=flore
```

The dashboard shows that the load is dispatched among the baristas.

# Instructions to run containers using Docker

1. Run Kafka
    ```bash
    docker-compose up
    ```
1. Build Docker Images
    * Barista-Kafka
    ```bash
    docker build -f barista-kafka/Dockerfile . -t barista-kafka
    ```
    * Barista-HTTP
    ```bash
    docker build -f barista-http/Dockerfile . -t barista-http
    ```
    * Coffeeshop-Service
    ```bash
    docker build -f coffeeshop-service/Dockerfile . -t coffeeshop-service
    ```
1. Run Docker Images
    * Barista-Kafka
    ```bash
    docker run -i --rm -p 8090:8090 --name barista-kafka_1 --network quarkus-coffeeshop-demo_default barista-kafka
    ```
    * Barista-HTTP
    ```bash
    docker run -i --rm -p 8082:8082 --name barista-http_1 --network quarkus-coffeeshop-demo_default barista-http
    ```
    * Coffeeshop-Service
    ```bash
    docker run -i --rm -p 8080:8080 --name coffeeshop-service_1 --network quarkus-coffeeshop-demo_default coffeeshop-service
    ```

# Instructions to run containers using Kubernetes

1. Build Docker Images
    * Barista-Kafka
    ```bash
    docker build -f barista-kafka/Dockerfile . -t barista-kafka
    ```
    * Barista-HTTP
    ```bash
    docker build -f barista-http/Dockerfile . -t barista-http
    ```
    * Coffeeshop-Service
    ```bash
    docker build -f coffeeshop-service/Dockerfile . -t coffeeshop-service
    ```
1. Run zookeeper and kafka server
    ```bash
    kubectl apply -f zookeeper-deployment.yaml
    kubectl apply -f zookeeper-service.yaml
    kubectl apply -f kafka-deployment.yaml
    kubectl apply -f kafka-service.yaml
    ```
1. Run services
    ```bash
    kubectl apply -f barista-kafka/deployment.yml
    kubectl apply -f barista-kafka/service.yml
    kubectl apply -f coffeeshop-service/deployment.yml
    kubectl apply -f coffeeshop-service/service.yml
    kubectl apply -f barista-http/deployment.yml
    kubectl apply -f barista-http/service.yml
    ```