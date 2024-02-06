# DevOps

TP1(/home/tp/DevOps/TP1) :

## **Database** Postgres server:

### Dockerfile:

```docker
FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```

### CMD Build:

```docker
docker build -t nogellex/db .
```

### Create Network:

```docker
docker network create app-network
```

### RUN without persistance:

```docker
docker run -p 15432:5432 --network app-network --name db nogellex/db
```

### Adminer:

```docker
docker run \
-p "8090:8080" \
--net=app-network \
--name=adminer \
-d \
adminer
```

### SQL files:

**CreateScheme.sql**

```sql
CREATE TABLE public.departments
(
 id      SERIAL      PRIMARY KEY,
 name    VARCHAR(20) NOT NULL
);

CREATE TABLE public.students
(
 id              SERIAL      PRIMARY KEY,
 department_id   INT         NOT NULL REFERENCES departments (id),
 first_name      VARCHAR(20) NOT NULL,
 last_name       VARCHAR(20) NOT NULL
);
```

**InsertData.sql**

```sql
INSERT INTO departments (name) VALUES ('IRC');
INSERT INTO departments (name) VALUES ('ETI');
INSERT INTO departments (name) VALUES ('CGP');

INSERT INTO students (department_id, first_name, last_name) VALUES (1, 'Eli', 'Copter');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Emma', 'Carena');
INSERT INTO students (department_id, first_name, last_name) VALUES (2, 'Jack', 'Uzzi');
INSERT INTO students (department_id, first_name, last_name) VALUES (3, 'Aude', 'Javel');
```

### Change dockerfile:

```docker
FROM postgres:14.1-alpine
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
COPY *.sql /docker-entrypoint-initdb.d
```

> rebuild and run
> 
- 1-1 Document your database container essentials: commands and Dockerfile.
    
    FROM : Create a new build stage from a base image.
    
    ENV : Set environment variables.
    
    COPY : Copy files and directories.
    
    ```docker
    docker build -t nogellex/db .
    docker network create app-network
    docker run -p 15432:5432 --network app-network --name db nogellex/db
    ```
    
    ⇒ Build, network creation and run 
    

### RUN with persistance:

```docker
docker run -p 15432:5432 --network app-network -v .//home/tp/DevOps/TP1/datasave:/var/lib/postgresql/data --name db nogellex/db
```

## **Backend API** JAVA :

**Main.java**

```java
public class Main {

   public static void main(String[] args) {
       System.out.println("Hello World!");
   }
}
```

```java
javac Main.java
```

### Simple api:

**GreetingController:**

```java
package fr.takima.training.simpleapi.controller;

import org.springframework.web.bind.annotation.*;

import java.util.concurrent.atomic.AtomicLong;

@RestController
public class GreetingController {

   private static final String template = "Hello, %s!";
   private final AtomicLong counter = new AtomicLong();

   @GetMapping("/")
   public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
       return new Greeting(counter.incrementAndGet(), String.format(template, name));
   }

   record Greeting(long id, String content) {}

}
```

**Dockerfile:**

```docker
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```

- 1-2 Why do we need a multistage build? And explain each step of this dockerfile.
    
    We need a multistage build to save  more ressources. 
    
    For exemple in a java code in a container :
    
    Multistage build will take a jdk and a jre, use the jdk to compile and then get rid of it to keep only the essential part witch is the .jar and the jre to execute it.
    
    Contrary to a classic build which will juste take and keep all the resources from build the execution.
    

```docker
docker build -t nogellex/java_d_api . -f s_api.dockerfile
docker run -p 8080:8080 --network app-network --name java_s_api nogellex/java_s_api
```

### **Backend API:**

**YML:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://db:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
  server:
    add-application-context-header: false
  endpoints:
    web:
      exposure:
        include: health,info,env,metrics,beans,configprops
```

```docker
docker build -t nogellex/java_b_api . -f b_api.dockerfile
docker run -p 8080:8080 --network app-network --name java_b_api nogellex/java_b_api
```

## **Http server :**

### Reverse proxy:

**httpd.conf:**

```docker
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://YOUR_BACKEND_LINK:8080/
ProxyPassReverse / http://YOUR_BACKEND_LINK:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

**Dockerfile:**

```java
FROM httpd:2.4
COPY ./httpd.conf /usr/local/apache2/conf/httpd-custom.conf
COPY ./public-html/ /usr/local/apache2/htdocs/
RUN echo "Include ./conf/httpd-custom.conf" >> /usr/local/apache2/conf/httpd.conf
```

**docker-compose:**

```yaml
version: '3.8'

services:
  backend:
    container_name: java_b_api
    build:
      context: TP1_2/simple-api-student-main
      dockerfile: b_api.dockerfile
    networks:
      - tp1
    depends_on:
      - database

  database:
    container_name: db
    build: TP1
    networks:
      - tp1
    volumes:
      - db:/var/lib/postgresql/data

  httpd:
    build: TP1_http
    ports:
      - 80:80
    networks:
      - tp1
    depends_on:
      - backend

networks:
	 tp1:
volumes:
	 db:
```

- 1-3 Document docker-compose most important commands.
    
    ```
    docker compose build
    ```
    
    => to build your containers and gather all informations necessary
    
    ```
    docker compose up -d
    ```
    
    => to wake up your builded container and run them
    
- 1-4 Document your docker-compose file.
    
    We use 3 services with 1 network and a volume for the database. Each service is link to a dockerfile, a context, a network and also if necessary a dependence.
    

**After you can build and up your docker :**

```bash
docker composer build && docker compose up -d
```

### Publish :

```bash
docker login -u nogellex
docker tag devops-database nogellex/tp1:1.0
docker push nogellex/tp1:1.0
```