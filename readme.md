# AppDev workshop

- 13:00 — 13:15 Developing with Spring Cloud          [Lecture]

- 13:15 — 13:45 Service Registration and Discovery    [Lecture]

- 13:45 — 14:15 Simple Discoverable applications      [Lab]

- 14:15 — 15:15 Service Discovery in the Cloud        [Lab]

- 15:15 — 15:45 Zero-Downtime Deployments for Discoverable services [Lab]

- 15:45 — 16:00 Break

- 16:00 — 16:30 Configuration Management              [Lecture]

- 16:30 — 17:00 Configuration Management in the Cloud [Lab]

- 17:00 — 17:15 Zuul                                  [Lecture / Lab]


<p>
<p>
## Developing with Spring Cloud

- Introduction to Spring Cloud and why it exists
- Spring Cloud OSS and Spring Cloud Services (PCF Tile)

## Spring Cloud Connectors

Cloud-agnostic library used to retrieve cloud services. In this context, cloud services mean the credentials to access those cloud service. Following the 12-factors: [Store config in the environment](https://12factor.net/config) and [Treat backing services as attached resources](https://12factor.net/backing-services). It is an SPI-based library where each cloud provider (e.g. Pivotal or Heroku) provide their own implementation. This project consists of 2 main modules: the core which only retrieves credentials from the cloud and a *service connectors* library that creates Spring beans of the appropriate type based on the discovered services.

In the next sections we will use some sample applications (labs/lab1 and labs/lab2) that declares the following dependencies in their pom.xml. This first dependency brings the *Connectors Core and Service Connectors* libraries (among others) and the second one brings the *Service Connectors for PCF*.
```
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-dependencies</artifactId>
  <version>Camden.SR4</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
<dependency>
  <groupId>io.pivotal.spring.cloud</groupId>
  <artifactId>spring-cloud-services-dependencies</artifactId>
  <version>1.4.1.RELEASE</version>
  <type>pom</type>
  <scope>import</scope>
</dependency>
```

The Cloud Foundry Connector discovers services bound to an application running in a Cloud Foundry environment. As it consumes bound service information in Cloud Foundry’s standard format, it is provider-agnostic; it currently is aware of application monitoring, DB2, MongoDB, MySQL, Oracle, PostgreSQL, RabbitMQ, Redis, SMTP, and SQL Server services.

## Applications and Services

### Consume Auto-configured Managed services
Cloud Foundry provides extensive support for connecting a Spring application to services such as MySQL, PostgreSQL, MongoDB, Redis, and RabbitMQ. In many cases, Cloud Foundry can automatically configure a Spring application without any code changes. In this case, Cloud Foundry automatically re-configures the relevant bean definitions to bind them to cloud services.

1. Create Mysql database service (managed service)
  `cf create-service p-mysql pre-existing-plan flight-repository`
2. Push application (with manifest, no jdbc driver)


But Cloud Foundry auto-reconfigures applications only if the following items are true for your application:
- Only one service instance of a given service type is bound to the application. In this context, different relational databases services are considered the same service type. For example, if both a MySQL and a PostgreSQL service are bound to the application, auto-reconfiguration does not occur.
- Only one bean of a matching type is in the Spring application context. For example, you can have only one bean of type javax.sql.DataSource.

### Consume Manually configured Managed services
Use manual configuration if you have multiple services of a given type or you want to have more control over the configuration than auto-reconfiguration provides.

We need to include the following dependency:
```
<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>
				spring-boot-starter-cloud-connectors
			</artifactId>
```
And add the logic that creates the `Datasource` in case of database services. The sample code shows how we can create a custom DataSource when we activate a profile.
```
@Configuration
	@Profile("custom")
	public class DataSourceConfig {
		@Primary
	    @Bean
	    public DataSource dataSource() {
	    	System.out.println("Created custom Datasource");
	        CloudFactory cloudFactory = new CloudFactory();
	        Cloud cloud = cloudFactory.getCloud();
	        List<ServiceInfo> serviceIDs = cloud.getServiceInfos(DataSource.class);
	        return cloud.getServiceConnector(serviceIDs.get(0).getId(), DataSource.class, null);
	    }
	}
```      
When we push this application, we need to activate the profile if we want to use the custom datasource.
We either manually set it `cf set-env flight-availability SPRING_PROFILES_ACTIVE custom` or thru the manifest.yml.




`cf set-env flight-availability SPRING_PROFILES_ACTIVE custom` or specify it in the manifest.yml.

### Non-Managed services such as Oracle Database
What happens if there is no service in the market place? Create CUPS and deal with it. Create a Cloud Connector Service Creator that is able to read the credentials from the CUPS.

## Service Registration and Discovery    [Lecture]

<a href="docs/SpringCloudServiceDiscovery.pdf">Slides</a>

- SCS gives the ability to have our applications talk each directly  without going thru the router. To do that we need have an application setting called  `spring.cloud.services.registrationMethod`. The values for this setting are : `route` and `direct`. If we use `route` (the default value), our applications register using their PCF route else if they register using their IP address.

```
NOTE: To enable direct registration, you must configure the PCF environment to allow traffic across containers or cells. In PCF 1.6, visit the Pivotal Cloud Foundry® Operations Manager®, click the Pivotal Elastic Runtime tile, and in the Security Config tab, ensure that the “Enable cross-container traffic” option is enabled.
```

##  Service Registration and Discovery    [Lab]

```
cf-demo-client ----{ http://demo/hi?name=Bob }--> cf-demo-app
                <----{ `hello Bob` }-------------
```

First we are going to get our 2 applications running locally. With our local Eureka server.
And the second part of the lab is to push our 2 applications to PCF and use PCF Service Registry to register our applications rather than our standalone Eureka server.

### Set up

1. You will need JDK 8, Maven and git. We will use the command line to launch the apps (`mvn spring-boot:run or java -jar`).
2. git clone https://github.com/MarcialRosales/spring-cloud-workshop

### Standalone Service Discovery

Go to the folder, `labs/lab1` in the cloned git repo.

1. Run eureka-server (from STS boot dashboard or from command line)
2. Go to the eureka-server url. This is the default eureka-server URL if we don't specify any.
`http://localhost:8761/`   
3. Run cf-demo-app
4. Check that our application registered with Eureka via the Eureka Dashboard.
5. Check that our app works
`curl localhost:8080/hello?name=Marcial`
6. Run cf-demo-client
7. Check that our application works, i.e. it automatically discover our demo app by its name and not by its url.

  `curl localhost:8081/hi?name=Bob`

  If we look at the source code `CfDemoClientApplication.ServiceInstanceRestController` we can see that we are using `http://demo/` as the base url when the real url is `http://localhost:8080`.
  ```
      ...
     @RequestMapping("/hi")
     public String hi(@RequestParam(defaultValue = "nobody") String name) {
       return restTemplate.exchange("http://demo/hello?name={name}",
         HttpMethod.GET, null,
         String.class, name).getBody();

     }
     ...
  ```

  The magic is provided by the annotation `@LoadBalanced` applied to a `RestTemplate` bean.
  ```
      @Bean
      @LoadBalanced   
      public RestTemplate restTemplate() {
      	return new RestTemplate();
      }
  ```
  The RestTemplate bean will be intercepted and auto-configured by Spring Cloud to use a custom HttpRequestClient that uses Netflix Ribbon to do the application lookup. Ribbon is also a load-balancer so if you have multiple instances of a service available, it picks one for you.
  The loadBalancer takes the logical service-name (as registered with the discovery-server) and converts it to the actual hostname of the chosen application.

8. Check that our application can discover services using the `DiscoveryClient` api.

  `curl localhost:8081/service-instances/demo | jq .`

9. stop the cf-demo-app
10. Check that it disappears from eureka but it is still visible to the client app.

  `curl localhost:8081/service-instances/demo | jq .`

  After 30 seconds it will disappear. This is because the client queries eureka every 30 seconds for a delta on what has happened since the last query.

11. stop eureka server, check in the logs of the demo app exceptions. Start the eureka server, and see that the service is restored, run to check it out:

  `curl localhost:8081/service-instances/demo | jq .`

Bonus labs:
- Run an additional instance of cf-demo-app (e.g. `mvn spring-boot:run -Dserver.port=9000`). Check Eureka dashboard and  `curl localhost:8081/service-instances/demo | jq .`. We should see 2 instances listed for DEMO app.
- See the `instanceId` attribute of each instance in `curl localhost:8081/service-instances/demo | jq .`. See that both instances have the same value `DEMO`. We can configure each instance with a unique identifier:
```
eureka:
  instance:
    metadataMap:
      instanceId: ${spring.application.name}:${spring.application.instance_id:${random.value}}
```
To run multiple instances of the same application (for load-balancing and resilience) they need to register with a unique id with the Registry Service. You will see that when we use Spring Cloud Connectors, Spring provides an unique identifier: `<appName>:<instanceIdentifier provided by PCF>`

We know our application works, we can push it to the cloud.

### Service Discovery in the Cloud

Note: Each attendee has its own account set up on this PCF foundation: https://apps.run-02.haas-40.pez.pivotal.io

**Create Eureka or Service Registry in PCF**

1. login
`cf login -a https://api.run-02.haas-40.pez.pivotal.io --skip-ssl-validation`

2. create service (http://docs.pivotal.io/spring-cloud-services/service-registry/creating-an-instance.html)

  `cf marketplace -s p-service-registry`

  `cf create-service p-service-registry standard registry-service`

  `cf services`

  Make sure the service is ready. PCF provisions the services asynchronously.

3. Go to the AppsManager and check the service. Check out the dashboard.

**Deploy cf-demo-app in PCF**

1. update manifest.yml (rename the host : eg. `yourname-cf-app-demo`, or `cf-app-demo-${random-word}`. )
2. push the application (check out url, domain)

  `cf push`

3. Check the app is working

  `curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/hello?name=Marcial`

4. Check the credentials PCF has injected into our application

  `cf env cf-demo-app`

  You will get something like this because we have stated in the `manifest.yml` that our application needs a service called `registry-service`. PCF takes care of injecting the credentials from the `registry-service` into the application's environment.

  ```
  {
   "VCAP_SERVICES": {
    "p-service-registry": [
     {
      "credentials": {
       "access_token_uri": "https://p-spring-cloud-services.uaa.system-dev.chdc20-cf.pez.com/oauth/token",
       "client_id": "p-service-registry-XXXXXXXX",
       "client_secret": "XXXXXXX",
       "uri": "https://eureka-044ac701-7919-4373-ad76-bdd0743fd813.apps-dev.chdc20-cf.pez.com"
      },
      "label": "p-service-registry",
      "name": "registry-service",
      "plan": "standard",
      "provider": null,
      "syslog_drain_url": null,
      "tags": [
       "eureka",
       "discovery",
       "registry",
       "spring-cloud"
      ],
      "volume_mounts": []
     }
    ]
   }
  }
  ```
  Thanks to a Spring project called [Spring Cloud Connectors ](http://cloud.spring.io/spring-cloud-connectors/) and [Spring Cloud Connectors for Cloud Foundry](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-cloud-foundry-connector.html), Spring Cloud Eureka is configured from the VCAP_SERVICES. If you want to know how you can start looking at [here](https://github.com/pivotal-cf/spring-cloud-services-connector/blob/master/spring-cloud-services-cloudfoundry-connector/src/main/java/io/pivotal/spring/cloud/cloudfoundry/EurekaServiceInfoCreator.java)

4. Go to the Dashboard  of the registry-service and check that our service is there
5. Scale the app to 2 instances. Check the dashboard.

**Deploy cf-demo-client in PCF**

1. install our client application
2. update manifest.yml (host)
3. push the application
4. Check the app is working

  `cf-demo-client.cfapps-02.haas-40.pez.pivotal.io/hi?name=Marcial`
5. Check that our app is not actually registered with Eureka however it has discovered our `demo` app. Also, see the instanceId shown in the Registry's dashboard.

6. We can rely on RestTemplate to automatically resolve a service-name to a url. But we can also use the Discovery API to get their urls.
`curl cf-demo-client.cfapps-02.haas-40.pez.pivotal.io/service-instances/demo | jq .`


### Eureka and dependency on Jersey 1.19. Path to Jersey 2.0.

- New Features on Jersey 2.0. Spring Web/REST vs Jersey 2.
- WIP eureka2 project based on Jersey 2.0 (https://github.com/Netflix/eureka/tree/master/eureka-client-jersey2)
- We still have to remove Ribbon transitive dependency on Jersey 1.19. It should be possible to remove it given that it has pluggable transport but it is a big job though.
- If we really want to leverage Netflix's load balancing capabilities the preferred path would be to keep working with Jersey 1 until Netflix updates all its stack to Jersey 2.


## Zero-Downtime Deployments for Discoverable services   [Lab]

We cannot register two PCF applications with the same `spring.application.name` against the same SCS `central-registry` service instance (but with different service's bindings or credentials) because according to SCS (1.1 and earlier) that is considered a security breached (i.e. another unexpected application is trying to register with the same name as another already registered application but using different credentials).

To go around this issue, we cannot bind PCF applications (blue and green) to the service instance of the service-registry (`p-service-registry`) because that will automatically create a new set of credentials for each application.

Instead, we need to ask the service instance -i.e. the `service-registry` from SCS- to provide us a credential and we create a `User Provided Service` with that credential. Once we have the `UPS` we can then bind that single `UPS` with our 2 applications, `green` and `blue`. That works because both instances, even though they are uniquely named in PCF they have the same `spring.application.name` used to register the app with Eureka and both apps are using the same credentials to talk to the `service-registry`, i.e. Eureka.

Go to the folder, labs/lab3 in the cloned git repo.

### Step by Step 1)
1. Create a new manifest and modify the attribute 'name' and change it to `cf-demo-app-green` and push the app.
2. It will fail because Eureka does not allow two PCF apps to register with Eureka using the same  `spring.application.name`.


### Step by Step 2)
1. Create a service instance of the service registry (skip this process if you already have a service instance)
   <br>`cf create-service p-service-registry standard central-registry`
2. Create a service key and call it `service-registry`
  <br>`cf create-service-key central-registry service-registry`
3. Read the actual key contained within the `service-registry` service key
  <br>`cf service-key central-registry service-registry`
  <br>It prints out something like this:

  ```
  Getting key service-registry for service instance central-registry as mrosales@pivotal.io...

{
 "access_token_uri": "https://p-spring-cloud-services.uaa.run.haas-35.pez.pivotal.io/oauth/token",
 "client_id": "p-service-registry-ce80e383-0691-4a0e-a48e-84df7035cb2e",
 "client_secret": "XXXXXXXX",
 "uri": "https://eureka-c890fdd0-18b5-4c5b-bc44-89ef2383dc08.cfapps.haas-35.pez.pivotal.io"
}
```

4. We have to create a `User Provided Service` with the credentials above. We will do that briefly.

5. We need to create a custom `EurekaServiceInfoCreator` class that is able to recognize our new `User Provided Service` as an Eureka Service. For reference, a `ServiceInfoCreator` is a Java class of the `spring cloud service connectors` library which is able to create a `ServiceInfo` from a `VCAP_SERVICES` variable. There are many types of services, for instances, databases, messaging middleware, you name it. For each type of service, there is a `ServiceInfoCreator` class. The `connectors` library has a list of those `ServiceInfoCreator` classes. During the bootstrap process, the `connectors` library iterates over the list of services declared in the `VCAP_SERVICES` variable. For each service, the `connectors` library asks each `ServiceInfoCreator` if they recognize that service as of its type. For instance, the  `EurekaServiceInfoCreator` will look up the value `eureka` in the `tags` attribute of the service. If there is a match, the `connectors` library asks the `EurekaServiceInfoCreator` to create an `EurekaServiceInfo` instance which later on it is used to configure the `Eureka client`.

We create a separate java project (`cf-demo-connectors`) for our custom `EurekaServiceInfoCreator` class so that we can bundle it with the `cf-demo-app` and `cf-demo-client` projects. Both applications will need to bind to the Eureka service therefore they need to find the eureka service in the `VCAP_SERVICES`.

6. We need to create a new (text) file that the `connectors` library use to identify `ServiceInfoCreator` classes in the class-path. This file must be located under `src/main/resources/META-INF/services/org.springframework.cloud.cloudfoundry.CloudFoundryServiceInfoCreator`. We add the following line to the file: `io.pivotal.demo.EurekaServiceInfoCreator`. We put this file in the project we created for the `EurekaServiceInfoCreator`.


7. Now we create a `User Provided Service` with the credentials above (Remember that we need to add our `label` attribute)
  <br>`cf cups service-registry -p '{"access_token_uri": "https://p-spring-cloud-services.uaa.run.haas-35.pez.pivotal.io/oauth/token","client_id": "p-service-registry-ce80e383-0691-4a0e-a48e-84df7035cb2e","client_secret": "WGE829u3U7qt","uri": "https://eureka-c890fdd0-18b5-4c5b-bc44-89ef2383dc08.cfapps.haas-35.pez.pivotal.io", "label": "eureka"}'`


8. Push your blue app : `cf push -f manifest.yml`
```
...
applications:
- name: cf-demo-app
  services:
  - service-registry
...
```

9. Repeat the process with green app: `cf push -f manifest-green.yml`
```
...
applications:
- name: cf-demo-app-green
  services:
  - service-registry
...
```

Both apps have `spring.application.name` equals to `demo`.

10. Check Eureka dashboard has one entry for our `demo` service with 2 urls, one for blue and another for green.


## 15:45 — 17:00 Configuration Management [Lecture]

[Slides](docs/SpringCloudConfigSlides.pdf)

Very interesting talk  [Implementing Config Server and Extending It](https://www.infoq.com/presentations/config-server-security)

### Additional comments
- We can store our credentials encrypted in the repo and Spring Config Server will decrypt them before delivering them to the client.
- Spring Config Service (PCF Tile) does not support server-side decryption. Instead, we have to configure our client to do it. For that we need to make sure that the java buildpack is configured with `Java Cryptography Extension (JCE) Unlimited Strength policy files`. For further details check out the <a href="http://docs.pivotal.io/spring-cloud-services/config-server/writing-client-applications.html#use-client-side-decryption">docs</a>.
- We should really use <a href="https://spring.io/blog/2016/06/24/managing-secrets-with-vault">Spring Cloud Vault</a> integrated with Spring Config Server to retrieve secrets.

## 15:45 — 17:00 Configuration Management  [Lab]
Go to the folder, labs/lab2 in the cloned git repo.

1. Check the config server in the market place
`cf marketplace -s p-config-server`
2. Create a service instance
`cf create-service -c '{"git": { "uri": "https://github.com/MarcialRosales/spring-cloud-workshop-config" }, "count": 1 }' p-config-server standard config-server`

3. Make sure the service is available (`cf services`)
3. Modify our application so that it has a `bootstrap.yml` rather than `application.yml`. We don't really need an `application.yml`. If we have one, Spring Config client will take that as the default properties of the application.

4. Our repo has already our `demo.yml`. If we did not have our the setting `spring.application.name`, the `spring-auto-configuration` jar injected by the java buildpack will automatically create a `spring.application.name` environment variable based on the env variable `VCAP_APPLICATION { ... "application_name": "cf-demo-app" ... }`.

5. Push our `cf-demo-app`.

6. Check that our application is now bound to the config server
`cf env cf-demo-app`

7. Check that it loaded the application's configuration from the config server.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/env | jq .`

We should have these configuration at the top :
```
"configService:https://github.com/MarcialRosales/spring-cloud-workshop-config/demo.yml": {
    "mymessage": "Good afternoon"
  },
  "configService:https://github.com/MarcialRosales/spring-cloud-workshop-config/application.yml": {
    "info.id": "${spring.application.name}"
  },
```  

8. Check that our application is actually loading the message from the central config and not the default message `Hello`.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/hello?name=Marcial`

9. We can modify the demo.yml in github, and ask our application to reload the settings.
`curl -X POST cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/refresh`

Check the message again.
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/hello?name=Marcial`


10. Add a new configuration for production : `demo-production.yml` to the repo.

11. Configure our application to use production profile by manually setting an environment variable in CF:
`cf set-env cf-demo-app SPRING_PROFILES_ACTIVE production`

we have to restage our application because we have modified the environment.

12. Check our application returns us a different value this type
`curl cf-demo-app.cfapps-02.haas-40.pez.pivotal.io/env | jq .`

We should have these configuration at the top :


## Configuration Management - Dynamically reconfigure your application [Lab]

### Dynamically changing settings at runtime in the application
- Spring will automatically rebuild a `@Bean` which is either annotated as `@RefreshScope` or `@ConfigurationProperties`. Spring assumes that a `ConfigurationProperties` is something we most likely want to refresh if any of its settings change.
- Spring will rebuild a bean when it receives a signal such as `RefreshRemoteApplicationEvent`. Spring config server sends this event to the application when it receives an event from the remote repo (via webhooks) or when it detects a change in the local filesystem -assuming we are using `native` repo.

Note about Reloading configuration: This works provided you only have one instance. Ideally, we want to configure our config server to receive a callback from Github (webhooks onto the actuator endpoint `/monitor`) when a change occurs. The config server (if bundled with the jar `spring-cloud-config-monitor`).
If we have more than one application instances you can still reload the configuration on all instances if you add the dependency `spring-cloud-starter-bus-amqp` to all the applications. It exposes a new endpoint called `/bus/refresh` . If we call that endpoint in one of the application instances, it propagates a refresh request to other instances.

Logging level is the most common settings we want to adjust at runtime. An Exercise is to modify the code to add a logger and add the logging level the demo.yml or demo-production.yml :
```
logging:
  level:
    io.pivotal.demo.CfDemoAppApplication: debug    

```


## Resiliency - What do we do if Config server is down?
We can either fail fast which means our applications fail to start. PCF would try to deploy the application a few times before giving up.
`spring.cloud.config.failFast=true`. Or if we can retry a few times before failing to start: `spring.cloud.config.failFast=false`, add the following dependencies to your project `spring-retry` and `spring-boot-starter-aop` and configure the retry mechanism via  `spring.cloud.config.retry.*`.

What happens if Config server cannot connect to the remote repo? @TODO test it!


## How to organize my application's configuration around the concept of a central repository [Lab]

#### Get started very quickly with spring config server : local file system (no git repo required)
```
---
spring.profiles: native
spring:
  cloud:
    config:
      server:
        native:
          searchLocations: ../../spring-cloud-workshop-config      
```

#### Use local git repo (all files must be committed!). One repo for all our applications and each application and profile has its own folder.
```
---
spring.profiles: git-local-common-repo
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../spring-cloud-workshop-config
          searchPaths: groupA-{application}-{profile}
```

#### Use local git repo. But different repos for different profiles
Spring Config server will try to resolve a pattern against ${application}/{profile}

```
---
spring.profiles: git-local-multi-repos-per-profile
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../emptyRepo
          repos:
            dev-repos:
              pattern: "*/dev"
              uri: file:../../dev-repo
            prod-repos:
              pattern: "*/prod"
              uri: file:../../prod-repo
```
 In this case, we have decided to have one repo specific for dev profile and another for prod profile              
 `curl localhost:8888/quote-service2/dev | jq .`

#### Use local git repo. Multiple repos per teams.

```
 ---
 spring.profiles: git-local-multi-repos-per-teams
 spring:
   cloud:
     config:
       server:
         git:
           uri: file:../../emptyRepo
           repos:
             trading:
               pattern: trading-*
               uri: file:../../trading
             pricing:
               pattern: pricing-*
               uri: file:../../pricing
             orders:
               pattern: orders-*
               uri: file:../../orders

```

We have 3 teams, trading, pricing, and orders. One repo per team responsible of a business capability.               
`curl localhost:8888/trading-execution-service/default | jq .`
`curl localhost:8888/pricing-quote-service/default | jq .`

#### Use local git repo. One repo per application.
```
---
spring.profiles: git-local-one-repo-per-app
spring:
  cloud:
    config:
      server:
        git:
          uri: file:../../{application}-repo
```          

## Zuul server [Lecture]

- @EnableZuulProxy
- It automatically (no configuration required) proxies all your services registered with Eureka thru a single entry point.
e.g. When the zuul proxy receives this request http://localhost:8082/demo/hello?name=Marcial it automatically forwards this request to http://localhost:8080/hello?name=Marcial
- We can configure Zuul to only allow certain services regardless of the services registered in Eureka. This is done thru simple configuration.
- However, we can customize Zuul internal behaviour. Zuul borrows the Servlet Filters concept from the Servlet specification. Every request is passed thru a number of filters and eventually the request is forwarded to destination, or not. The filters allow us to intercept requests at different stages: before the request is routed, after we receive a response from the destination service. There are special type of filters which we can use to override the routing logic.

### Build a Zuul server [Lab]

The source code for this lab is available under `labs\lab4`. This lab relies on the previous lab3 artifacts, i.e. `cf-demo-app`, `eureka-server` (if you run it locally else Eureka from SCS) and `config-server` (if you run it locally else Config server from SCS). It also relies on the configuration file `demo-gateway.yml` in the configuration repository.

1. To create a Zuul server we simply create one like this:
```
@EnableZuulProxy
@SpringBootApplication
public class GatewayServiceApplication {

	Map<String, Object> basicCache = new ConcurrentHashMap<>();

	@Bean
	public ZuulFilter histogramAccess(RouteLocator routeLocator, MetricRegistry metricRegistry) {
		return new StatsCollector(routeLocator, metricRegistry);
	}
	public static void main(String[] args) {
		SpringApplication.run(GatewayServiceApplication.class, args);
	}
}
```
2. We implement our own filter which keeps track of number of requests per service:
```
class StatsCollector extends ZuulFilter {

	private static Logger log = LoggerFactory.getLogger(StatsCollector.class);
	private MetricRegistry metrics;
	private Map<String,String> serviceAliases = new HashMap<>();

	public StatsCollector(RouteLocator routeLocator, MetricRegistry registry) {
		super();
		this.metrics = registry;
		routeLocator.getRoutes().forEach(r -> {
			String alias = aliasForService(r.getLocation());
			serviceAliases.put(r.getLocation(), alias);
			metrics.counter(alias);			
		});
	}
	private String aliasForService(String name) {
		return String.format("metrics.%s.requestCount", name);
	}

	@Override
	public boolean shouldFilter() {
		return true;
	}

	@Override
	public int filterOrder() {
		return 10;
	}

	@Override
	public String filterType() {
		return "pre";
	}

	@Override
	public Object run() {
		RequestContext ctx = RequestContext.getCurrentContext();

		metrics.counter(serviceAliases.get((String)ctx.get("serviceId"))).inc();

		return null;
	}

}
```

3. And we configure it so that all requests must be prefixed with `/api`. Also we want to disable every service registered with Eureka except our `cf-demo-app`. We can use a different name for our service in the URL. Instead of  `/api/demo/` but use `/api/demo-service`.
```

zuul:
  prefix: /api
  ignored-services: '*'
  routes:
    demo: /demo-service/**
```    

4. Invoke the service several times (eg. `http://localhost:8082/api/demo-service/hello?name=Bob`) and check the metrics using the metrics endpoint `http://localhost:8082/metrics | jq '.["metrics.demo.requestCount"]' `.
