# deors.demos.microservices.docs

## microservices with Spring Boot and Spring Cloud

this repository contains the step-by-step instructions to recreate the microservices with Spring Boot and Spring Cloud demonstration

this demo is organised in iterations, starting from the basics and building up in complexity and features along the way

NOTE: the following instructions are created on a Windows machine - some commands may need slight adjustments when working on Linux/OSX

## iteration 1) the basics

### 1.1) set up the configuration store

the configuration store is a repository where microservice settings are stored, and accessible for microservice initialisation at boot time

create and change to a directory for the project

    mkdir %HOME%\microservices\deors.demos.microservices.configstore
    cd %HOME%\microservices\deors.demos.microservices.configstore

create file application.properties; these settings are common to all microservices

    debug = true
    spring.jpa.generate-ddl = true

create file eureka-service.properties

    server.port = ${PORT:7878}
    eureka.client.register-with-eureka = false
    eureka.client.fetch-registry = false

create file hystrix-dashboard.properties

    server.port = ${PORT:7979}

create file bookrec-service.properties

    server.port = ${PORT:8080}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/

create file bookrec-edgeservice.properties

    server.port = ${PORT:8181}
    eureka.client.serviceUrl.defaultZone = http://${EUREKA_HOST:localhost}:${EUREKA_PORT:7878}/eureka/
    ribbon.eureka.enabled = true
    defaultBook = this is the default recommendation: Book {id=-1, title='robots of dawn', author='isaac asimov'}

initialise the Git repository

    git init
    git add .
    git commit -m "initial configuration"

publish it online (i.e. GitHub, replace with your own repository)

    git remote add origin https://github.com/deors/deors.demos.microservices.configstore.git
    git push origin master

### 1.2) set up the configuration server

the configuration server is the microservice that will provide every other microservice in the system with the configuration settings they need at boot time

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: configservice
    depedencies: config server

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\configservice

ensure config store location is properly set;
edit src\main\resources\application.properties

    server.port = ${PORT:8888}
    spring.cloud.config.server.git.uri = ${CONFIG_VOL:https://github.com/deors/deors.demos.microservices.configstore.git}

configure config server to start automatically;
edit src\main\java\deors\demos\microservices\configservice\ConfigserviceApplication.java

add class annotation

    @org.springframework.cloud.config.server.EnableConfigServer

### 1.3) set up the service registry server (Eureka)

the service registry server is the microservice that will enable every other microservice in the system to register 'where' they are physically located, so others can discover them and interact with them

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: eurekaservice
    depedencies:
        eureka server
        config client

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\eurekaservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = eureka-service
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure eureka server to start automatically;
edit src\main\java\deors\demos\microservices\eurekaservice\Application.java

add class annotation

    @org.springframework.cloud.netflix.eureka.server.EnableEurekaServer

### 1.4) set up the short-circuit dashboard (Hystrix)

the short-circuit dashboard will provide devs and ops teams with real-time views about service interactions including for example which of them are experimenting failures causing circuits to 'open'

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: hystrixdashboard
    depedencies:
        hystrix dashboard
        eureka discovery
        config client

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\hystrixdashboard

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = hystrix-dashboard
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure hystrix dashboard to start automatically;
edit src\main\java\deors\demos\microservices\hystrixdashboard\HystrixdashboardApplication.java

add class annotation

    @org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard

### 1.5) set up the book recommendation service

this is the first microservice with actual functionality on our problem domain; bookrec is the service which provides methods to query, create, update and remove Book entities from the data store

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: bookrecservice
    depedencies:
        eureka discovery
        config client
        jpa
        rest repositories
        rest repositories hal browser
        hateoas
        web
        h2

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\bookrecservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = bookrec-service
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure service to be discoverable;
edit src\main\java\deors\demos\microservices\bookrecservice\BookrecserviceApplication.java

add class annotation

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient

create Book entity (in IDE)

    @Entity
    public class Book {
        @Id
        @GeneratedValue
        private Long id;
        private String title;
        private String author;
    }

generate bean constructors, getters, setters and toString method

create BookRepository

    @RepositoryRestResource
    public interface BookRepository extends CrudRepository<Book, Long> {
        @Query("select b from Book b order by RAND()")
        List<Book> getBooksRandomOrder();
    }

create BookController

    @RestController
    public class BookController {
        @Autowired
        private BookRepository bookRepository;
        @RequestMapping("/bookrec")
        public String getBookRecommendation() throws UnknownHostException {
            return "the host in IP: "
                    + InetAddress.getLocalHost().getHostAddress()
                    + " recommends this book: "
                    + bookRepository.getBooksRandomOrder().get(0);
        }
    }

add some test data (in IDE, create src/main/resources/import.sql)

    insert into book(id, title, author) values (1, 'second foundation', 'isaac asimov')
    insert into book(id, title, author) values (2, 'speaker for the dead', 'orson scott card')
    insert into book(id, title, author) values (3, 'the player of games', 'iain m. banks')
    insert into book(id, title, author) values (4, 'the lord of the rings', 'j.r.r. tolkien')
    insert into book(id, title, author) values (5, 'the warrior apprentice', 'lois mcmaster bujold')
    insert into book(id, title, author) values (6, 'blood of elves', 'andrzej sapkowski')
    insert into book(id, title, author) values (7, 'harry potter and the prisoner of azkaban', 'j.k. rowling')
    insert into book(id, title, author) values (8, '2010: odyssey two', 'arthur c. clarke')
    insert into book(id, title, author) values (9, 'starship troopers', 'robert a. heinlein')

### 1.6) set up the book recommendation edge service

the bookrec edge service is used by clients to interact with bookrec service, which should be not exposed directly to clients

go to https://start.spring.io/

create project

    group: deors.demos.microservices
    artifact: bookrecedgeservice
    depedencies:
        eureka discovery
        config client
        hystrix
        ribbon
        zuul
        rest repositories hal browser
        hateoas
        web

extract zip to

    %HOME%\microservices

change into extracted directory

    cd %HOME%\microservices\bookrecedgeservice

ensure config service is used by moving props to bootstrap phase

    ren src\main\resources\application.properties bootstrap.properties

edit src\main\resources\bootstrap.properties

    spring.application.name = bookrec-edgeservice
    spring.cloud.config.uri = http://${CONFIG_HOST:localhost}:${CONFIG_PORT:8888}

configure service to be discoverable and to use zuul proxy;
edit src\main\java\deors\demos\microservices\bookrecedgeservice\BookrecedgeserviceApplication.java

add class annotations

    @org.springframework.cloud.client.discovery.EnableDiscoveryClient
    @org.springframework.cloud.netflix.zuul.EnableZuulProxy

add restTemplate() method to enable load balancing when calling bookrec-service

    @Bean
    @org.springframework.cloud.client.loadbalancer.LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

create Book bean (in IDE)

    public class Book {
        private Long id;
        private String title;
        private String author;
    }

generate bean constructors (one with three properties), getters, setters and toString method

create BookController for edge service

    @RestController
    class BookController {
        @Autowired
        RestTemplate restTemplate;

        @Value("${defaultBook}")
        private String defaultBook;

        @RequestMapping("/bookrecedge")
        @com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand(fallbackMethod = "getDefaultBook")
        public String getBookRecommendation() {
            return restTemplate.getForObject("http://bookrec-service/bookrec", String.class);
        }

        public String getDefaultBook() {
            return defaultBook;
        }
    }

### 1.7) running services locally

services are now ready to be executed locally, using the sensible default configuration settings and the embedded runtimes provided by Spring Boot

each service will be run by executing this command in the project folder:

    mvn spring-boot:run

### 1.8) test services locally

the services should all be available at the defined ports in the local host

access the config service

    http://localhost:8888/env

check that config service is capable of returning the right configuration for some of the services

    http://localhost:8888/bookrec-service/default
    http://localhost:8888/eureka-service/default

check that eureka service is up and the book recommendation service and edge service are registered

    http://localhost:7878/

check that hystrix dashboard is up and running

    http://localhost:7979/hystrix

access the HAL browser on the book recommendation service

    http://localhost:8080/

access the book recommendation service

    http://localhost:8080/bookrec

access the book recommendation edge service

    http://localhost:8181/bookrecedge

stop the book recommendation service, and access the book recommendation edge service again;
the default recommended book should be returned instead but the application keeps working

## iteration 2) preparing for Docker and Swarm

### 2.1) setting up a swarm

this demo assumes that an existing Docker Swarm is available; the following instructions will show how to create a simple one in VirtualBox

the swarm will be formed by three manager nodes, and three worker nodes, named:

    docker-swarm-manager-1
    docker-swarm-manager-2
    docker-swarm-manager-3
    docker-swarm-worker-1
    docker-swarm-worker-2
    docker-swarm-worker-3

the machines will be deployed in its own network:

    192.168.88.1/24

being the first IP in DHCP pool:

    192.168.88.100

to create each machine in VirtualBox, this is the command that was used

    docker-machine create --driver virtualbox --virtualbox-cpu-count 1 --virtualbox-memory 1024 --virtualbox-hostonly-cidr "192.168.88.1/24" <docker-machine-name>

once they are created, the swarm must be initialized;
but before, set the environment to point to the first machine (docker-env is a handy script to issue the FOR command needed in Windows)

    docker-env docker-swarm-manager-1
    docker swarm init --advertise-addr 192.168.88.100

upon initialization, the swarm exposes two tokens, one to add new manager nodes, one to add new worker nodes;
the commands needed to get the tokens are these ones

    docker swarm join-token manager -q
    docker swarm join-token worker -q

with the tokens, just move the environment to each machine, managers and workers, and issue this command

    docker-env <docker-machine-name>
    docker swarm join --token <manager-or-worker-token> 192.168.88.100:2377

once the swarm is created, it can be stopped and started again with the following command sets; to stop the swarm:

    docker-machine stop docker-swarm-manager-1
    docker-machine stop docker-swarm-manager-2
    docker-machine stop docker-swarm-manager-3
    docker-machine stop docker-swarm-worker-1
    docker-machine stop docker-swarm-worker-2
    docker-machine stop docker-swarm-worker-3

to start it again:

    docker-machine start docker-swarm-manager-1
    docker-machine start docker-swarm-manager-2
    docker-machine start docker-swarm-manager-3
    docker-machine start docker-swarm-worker-1
    docker-machine start docker-swarm-worker-2
    docker-machine start docker-swarm-worker-3

### 2.2) dockerize the services

configure pom.xml and Dockerfile to allow each service to run as a Docker image

    cd %HOME%\microservices\bookrecservice

edit pom.xml

add inside &lt;properties&gt;

    <docker.image.prefix>deors</docker.image.prefix>

add inside &lt;build&gt;&lt;plugins&gt;

    <plugin>
        <groupId>com.spotify</groupId>
        <artifactId>docker-maven-plugin</artifactId>
        <version>0.4.11</version>
        <configuration>
            <imageName>${docker.image.prefix}/${project.groupId}.${project.artifactId}</imageName>
            <dockerDirectory>src/main/docker</dockerDirectory>
            <imageTags>
                <imageTag>${project.version}</imageTag>
                <imageTag>latest</imageTag>
            </imageTags>
            <serverId>docker-hub</serverId>
            <resources>
                <resource>
                    <targetPath>/</targetPath>
                    <directory>${project.build.directory}</directory>
                    <include>${project.build.finalName}.jar</include>
                </resource>
            </resources>
        </configuration>
    </plugin>

create directory for Dockerfile

    mkdir src\main\docker

edit src\main\docker\Dockerfile

    FROM frolvlad/alpine-oraclejdk8:slim
    VOLUME /tmp
    ADD bookrecservice-0.0.1-SNAPSHOT.jar app.jar
    ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

repeat for the other projects (don't forget to update Jar file name in ADD command)

### 2.3) creating the images

launch swarm (one machine is enough)

    docker-machine start docker-swarm-manager-1
    docker-machine start docker-swarm-manager-2
    docker-machine start docker-swarm-manager-3
    docker-machine start docker-swarm-worker-1
    docker-machine start docker-swarm-worker-2
    docker-machine start docker-swarm-worker-3

point docker client to one of the machines

    docker-env docker-swarm-manager-1

build and push images by running this command for each project

    mvn package docker:build -DpushImage

### 2.4) running the images as services

create an overlay network for all the services

    docker network create -d overlay microdemo-network

launch config-service and check the status

    docker service create -p 8888:8888 --name config-service --network microdemo-network deors/deors.demos.microservices.configservice:latest
    docker service ps config-service

launch eureka-service and check the status

    docker service create -p 7878:7878 -e "CONFIG_HOST=config-service" --name eureka-service --network microdemo-network deors/deors.demos.microservices.eurekaservice:latest
    docker service ps eureka-service

launch hystrix-dashboard and check the status

    docker service create -p 7979:7979 -e "CONFIG_HOST=config-service" --name hystrix-dashboard --network microdemo-network deors/deors.demos.microservices.hystrixdashboard:latest
    docker service ps hystrix-dashboard

launch bookrec-service and check the status

    docker service create -p 8080:8080 -e "CONFIG_HOST=config-service" -e "EUREKA_HOST=eureka-service" --name bookrec-service --network microdemo-network deors/deors.demos.microservices.bookrecservice:latest
    docker service ps bookrec-service

launch bookrec-edgeservice and check the status

    docker service create -p 8181:8181 -e "CONFIG_HOST=config-service" -e "EUREKA_HOST=eureka-service" --name bookrec-edgeservice --network microdemo-network deors/deors.demos.microservices.bookrecedgeservice:latest
    docker service ps bookrec-edgeservice

### 2.5) test services in the swarm

access the config service

    http://192.168.88.100:8888/env

check that config service is capable of returning the right configuration for some of the services

    http://192.168.88.100:8888/bookrec-service/default
    http://192.168.88.100:8888/eureka-service/default

check that eureka service is up and the book recommendation service and edge service are registered

    http://192.168.88.100:7878/

check that hystrix dashboard is up and running

    http://192.168.88.100:7979/hystrix

access the HAL browser on the book recommendation service

    http://192.168.88.100:8080/

access the book recommendation service

    http://192.168.88.100:8080/bookrec

access the book edge recommendation service

    http://192.168.88.100:8181/bookrecedge

stop the book recommendation service, and access the book recommendation edge service again;
the default recommended book should be returned instead but the application keeps working

### 2.6) scale the book recommendation service

ask Docker to scale the service

    docker service scale bookrec-service=3

### 2.7) make updates and roll the changes without service downtime

make some change and deploy a rolling update;
for example change text string in BookController class stored here: src\main\java\deors\demos\microservices\BookController.java

rebuild and push the new image to the registry

    mvn package docker:build -DpushImage

deploy the change;
a label is needed to ensure the new version of 'latest' image is downloaded from registry

    docker service update --container-label-add update_cause="change" --update-delay 30s --image deors/deors.demos.microservices.bookrecservice:latest bookrec-service

check how the change is deployed

    docker service ps bookrec-service

## appendixes

### clean up the swarm

remove running services

    docker service rm config-service
    docker service rm eureka-service
    docker service rm hystrix-dashboard
    docker service rm bookrec-service

verify they are all removed

    docker service ls

remove all stored images (this must be done in every machine if more than one was used)

if in windows

    for /F %f in ('docker ps -a -q') do (docker rm %f)
    for /F %f in ('docker images -q') do (docker rmi --force %f)

if in linux

    docker rm $(docker ps -a -q)
    docker rmi --force $(docker images -q)

stop the machines

    docker-machine stop docker-swarm-worker-3
    docker-machine stop docker-swarm-worker-2
    docker-machine stop docker-swarm-worker-1
    docker-machine stop docker-swarm-manager-3
    docker-machine stop docker-swarm-manager-2
    docker-machine stop docker-swarm-manager-1

### troubleshooting

if using more than one machine in the swarm, images must be published to Docker Hub so they are accessible to all hosts in the swarm

if needed to troubleshoot connectivity with curl in alpine-based images, install and use this way

    docker exec <id> apk add --update curl && curl <url>
