# Kong Demo
Demonstrating cannary deployment using Kong API plugins 


## Install Kong (using Docker)
`docker network create kong-net`

```
docker run -d --name kong-database \
    --network=kong-net \
    -p 5432:5432 \
    -e "POSTGRES_USER=kong" \
    -e "POSTGRES_DB=kong" \
    postgres:9.6
```
               
```docker run --rm \
    --network=kong-net \
    -e "KONG_DATABASE=postgres" \
    -e "KONG_PG_HOST=kong-database" \
    -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
    kong:latest kong migrations bootstrap
```


Follow the link below to download a preconfigured Spring Boot application using Java 8 and Maven. The link prefills in depedancies on Spring Web, JPA, Rest Repositories, H2 Embedded DB, and Actuator. These are common Spring projects used in enterpirse microservices.

[Spring Initializr](https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.2.0.RELEASE&packaging=jar&jvmVersion=1.8&groupId=kong&artifactId=canary-demo&name=canary-demo&description=Demonstrating%20canary%20deployment%20using%20Kong%20API%20plugins%20&packageName=kong.canary-demo&dependencies=data-rest,web,data-jpa,h2,actuator "Includes Spring Web, JPA, REST Repo, and Actuator")

