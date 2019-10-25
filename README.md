![Kong API](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/konglogo.svg "Kong API Manager")
# Kong Demo
Demonstrating cannary deployment using Kong API plugins 


## Install Kong (using Docker) ![Docker](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/docker.png "Docker")


Using the below steps install Kong using a PostgresDB \[[Documentation](https://docs.konghq.com/install/docker/?_ga=2.202727505.1528094160.1571771527-1072143887.1570723420)\]


1. Create a custom docker network

   `docker network create kong-net`

2. Install the Postgres DB Container on the network created in step 1

   ```
   docker run -d --name kong-database \
     --network=kong-net \
     -p 5432:5432 \
     -e "POSTGRES_USER=kong" \
     -e "POSTGRES_DB=kong" \
     postgres:9.6
   ```
               
3. Perform the data migration from Kong latest

   ```docker run --rm \
     --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     kong:latest kong migrations bootstrap
   ```
4. Start Kong using the latest tag and these properties

   ```
   docker run -d --name kong \
     --network=kong-net \
     -e "KONG_DATABASE=postgres" \
     -e "KONG_PG_HOST=kong-database" \
     -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 8001:8001 \
     -p 8444:8444 \
     kong:latest
   ```
5. Verify Kong by cURL against the admin API at 8001

   `curl -i http://localhost:8001/`

## Spring Boot Application

For the service we will launch a simple "Hello world" Spring Boot Application.

Follow the link below to download a preconfigured Spring Boot application using Java 8 and Maven. The link prefills in depedancies on Spring Web, JPA, Rest Repositories, H2 Embedded DB, and Actuator. These are common Spring projects used in enterpirse microservices.

[Spring Initializr](https://start.spring.io/#!type=maven-project&language=java&platformVersion=2.2.0.RELEASE&packaging=jar&jvmVersion=1.8&groupId=kong&artifactId=canary-demo&name=canary-demo&description=Demonstrating%20canary%20deployment%20using%20Kong%20API%20plugins%20&packageName=kong.canary-demo&dependencies=data-rest,web,data-jpa,h2,actuator "Includes Spring Web, JPA, REST Repo, and Actuator")

In your favorite IDE open the application's main class `CanaryDemoApplication.java`. We need to annotate the class to function as a REST Controller. Below `@SpringBootApplication` add `@RestController`. Inside the class, below the main method add the following "Hello world" API.

```java
@RequestMapping("/")
public String hello() {
   return "Hello World!";
}
```

Now we can use maven wrapper to build the application then run it

```
./mvnw clean package
java -jar target/canary-demo-0.0.1-SNAPSHOT.jar
```

At the very end you should see output similar to below, showing our applicaiton is being served at `localhost:8080`. This is the default host and port for the embedded Tomcat server which is bundled in our Spring Boot application by way of the Spring Web project we included in our initializr.

```
2019-10-24 03:34:03.336  INFO 10411 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2019-10-24 03:34:03.339  INFO 10411 --- [           main] kong.canarydemo.CanaryDemoApplication    : Started CanaryDemoApplication in 4.404 seconds (JVM running for 4.818)
```

We can verify our app is up and runing by either curl or navigating the browser to `http://localhost:8080/` and you will see

`Hello World!`

Since we also added the Actuator module so navigate to `http://localhost:8080/actuator` to get an output of the Actuator mappings available.

Using Pivotal Cloud Foundry, we need to push our application and get a route.

```
cf push canary-demo -p target/canary-demo-0.0.1-SNAPSHOT.jar
...
routes:            canary-demo.cfapps.io
```


## Configure Kong

At this point we have the Kong API Manager running in Docker with our Hello World Spring Boot Application running at `canary-demo.cfapps.io`. Now we need to configure a Service object for the application. \[[Documentation](https://docs.konghq.com/1.3.x/getting-started/configuring-a-service/)\]

To do this we are going to leverage [Insomnia Rest Client](https://insomnia.rest/ "Now part of Kong!"). This will allow us to save the requests we issue to make troubleshooting much easier. From the official documenation we will need to convert from cURL syntax to JSON, which we have provided below.

1. Add the canary service and point it to our applicaiton

```
   curl -i -X POST \
  --url http://localhost:8001/services/ \
  --data 'name=canary-service' \
  --data 'url=canary-demo.cfapps.io'
```
![Canary-Service](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-canary-service.png)

2. Next add a route for the service we just created. This will be used in future requests to provide a vanity URL for our application

```
curl -i -X POST \
  --url http://localhost:8001/services/canary-service/routes \
  --data 'hosts[]=canary.com'
```
![Canary-Route](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-canary-route.png)

3. Verify the request is being forwarded by issuing a cURL request against Kong's public entry point referferncing the URL we set in the previous step

```
curl -i -X GET \
  --url http://localhost:8000/ \
  --header 'Host: canary.com'
```
![Canary-Get](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/get-canary.png)

## Plugins

At this point we have the basic Serivice gateway from Kong to our app running in the cloud. Now we are going to use Kong's Plugin library to apply behaviour to our requests. \[[Documentation](https://docs.konghq.com/hub/?_ga=2.126640589.1528094160.1571771527-1072143887.1570723420)\]

### [Authorization](https://docs.konghq.com/hub/kong-inc/key-auth/)

As with all Kong configuration, issue a cURL request to enable the key-auth plugin
```
curl -i -X POST \
  --url http://localhost:8001/services/canary-service/plugins/ \
  --data 'name=key-auth'
```
![Key-Auth](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-key-auth.png)

Making a request to our service will now return a `401 Unauthorized` so now that we have a lock, lets make a key.

#### Consumers

Issue a cURL request to create a Consumer and key credentials
```
curl -i -X POST \
  --url http://localhost:8001/consumers/ \
  --data "username=Vince"
```
![Consumer](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-consumer.png)
```
curl -i -X POST \
  --url http://localhost:8001/consumers/Vince/key-auth/ \
  --data 'key=reallyStrongPassword'
```
![Api-Key](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-api-key.png)

Now we add our apikey to our header and voila, we have access again.
```
curl -i -X GET \
  --url http://localhost:8000 \
  --header "Host: canary.com" \
  --header "apikey: reallyStrongPassword"
```
![Authorized](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/authorized.png)

### [Rate Limiting](https://docs.konghq.com/hub/kong-inc/rate-limiting/)

The last thing we want to add is a simple rate limiter that will throttle the requests made to our Service. Once again a simple cURL request will configure the plugin. We are applying this plugin directly to the Service, however it can be set on any Kong objects (consumer, routes, etc.) (Please see documentation for more detail)

```
curl -X POST http://kong:8001/services/canary-service/plugins \
  --data "name=rate-limiting"  \
  --data "config.second=5" \
  --data "config.minute=10" \
  --data "config.hour=10000" \
```
![Rate-Limit](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/post-rate-limit.png)
![Show-Rate-Limit](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/show-rate-limiting.png)

Once applied you can reload the page and check the header to see the current request period, requests remaining and the timeouts of each setting. Use `curl -X PATCH` to change the rate-limiter configuration.
![Rate-Limited](https://github.com/vrusso-pivotal/kong-canary-demo/blob/master/assets/rate-limiting.png)
