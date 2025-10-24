# Part 1
## Database
### Basics
```
$ docker build -t yuiko911/postgres
[...]
Successfully built 072c46f43171
Successfully tagged yuiko911/postgres:latest
```

We create a network to isolate our containers from the outside network, and have them be able to connect to each other easily. Then, we start the Postgres container on this network, in detached mode.
```
$ docker network create app-network 
$ docker run -d --network app-network yuiko911/postgres:latest 
```

Running Adminer in detached mode.
```
$ docker run -d --network app-network -p 8080:8080 adminer 
```
*Note: In the "Server" field of Adminer, we should not put "localhost", but rather the name of the container, in this case "epic_wilbur"*

### Question 1-1
**For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?** \
Environment variables often contain sensitive information, such as passwords. Putting a password in the Dockerfile is insecure, as it is written directly in easily recognisable plain text, accesible for an attacker to search, find and see it. Using -e, that risk is minimized, as it's only written once on lauch (though the risk is still present if the attacker has acces to the command history) 

We can remove the environment variables from the Dockerfile...
```
$ docker rm -f epic_wilbur
$ docker image rm yuiko911/postgres:latest
# Removing the environment variables from the Dockerfile
$ docker build -t yuiko911/postgres .
[...]
Successfully built 2bf60600670f
Successfully tagged yuiko911/postgres:latest
```
...and then run the container again.
```
$ docker run -d --name postgres_con \
	--network app-network \
	-e POSTGRES_DB=tp_db \
	-e POSTGRES_USER=tp_usr \
    -e POSTGRES_PASSWORD=tp_pwd \
 	yuiko911/postgres:latest 
```

### Init database
To have initial data, we add the following commands to the Dockerfile :
```dockerfile
COPY ./scripts/* /docker-entrypoint-initdb.d/
USER root
RUN chown postgres:postgres /docker-entrypoint-initdb.d/*.sql
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]
```

Then, we rebuild the image...
```
$ docker rm -f postgres_con
$ docker image rm yuiko911/postgres:latest
$ docker build -t yuiko911/postgres .
[...]
Successfully built fe5853b5df8c
Successfully tagged yuiko911/postgres:latest
```

...and restart the services.
```
$ docker run -d --name postgres_con \         
	--network=app-network \
	-e POSTGRES_DB=tp_db \
	-e POSTGRES_USER=tp_usr \
    -e POSTGRES_PASSWORD=tp_pwd \
	yuiko911/postgres:latest
$ docker run -d --name=adminer \
    --net=app-network \
    -p "8090:8080" \
    adminer
```
We now see the database correctly populated with data.

### Persist data
```
$ docker run -d --name=postgres_con \         
	--network=app-network \
	-e POSTGRES_DB=tp_db \
	-e POSTGRES_USER=tp_usr \
    -e POSTGRES_PASSWORD=tp_pwd \
	-v /home/yuuka/Documents/EFREI/I2-Docker/TP1/Postgres-data:/var/lib/postgresql/data \
	yuiko911/postgres:latest
```

Data is present on launch, we can then try to create a test table from Adminer.
We restart the container :
```
$ docker rm -f postgres_con
$ docker run -d --name=postgres_con --network=app-network -e POSTGRES_DB=tp_db -e POSTGRES_USER=tp_usr -e POSTGRES_PASSWORD=tp_pwd -v /home/yuuka/Documents/EFREI/I2-Docker/TP1/Postgres-data:/var/lib/postgresql/data yuiko911/postgres:latest
```
And we can see that the test table is indeed still present. \
*Note: The Postgres-data folder now has no permission for user outside docker, so we can't access it by default without using chmod*

### Question 1.2
**Why do we need a volume to be attached to our postgres container?** \
We need a volume so that the database's content is not reset each time we restart the container. It also allows us to more easily migrate it if needed. 

### Question 1.3
**Document your database container essentials: commands and Dockerfile.** \

```dockerfile
# Dockerfile
FROM postgres:17.2-alpine

COPY ./scripts/* /docker-entrypoint-initdb.d/
USER root
RUN chown postgres:postgres /docker-entrypoint-initdb.d/*.sql
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["postgres"]
``` 
```
# Building the image
$ docker built -t yuiko911/postgres .

# Running Postgres
$ docker run -d --name=postgres_con --network=app-network -e POSTGRES_DB=tp_db -e POSTGRES_USER=tp_usr -e POSTGRES_PASSWORD=tp_pwd -v /home/yuuka/Documents/EFREI/I2-Docker/TP1/Postgres-data:/var/lib/postgresql/data yuiko911/postgres:latest

# Running Adminer
$ docker run -d --name=adminer --net=app-network -p "8090:8080" adminer
```

## Backend API
### Basics
```
# Making sure to use Java 21 to compile
$ javac Main.java
$ docker built -t yuiko911/backendapi .
Successfully built 236ddaad83f5
Successfully tagged yuiko911/backendapi:latest
$ docker run yuiko911/backendapi:latest  
Hello World!
```

```
$ docker build -t yuiko911/simpleapi .
$ docker run -d --name=simple_api --net=app-network -p 8080:8080 yuiko911/simpleapi:latest
```
*Note: Make sure to not use the same port as another container (Adminer in my case)*

The URL scheme for the database is `jdbc:postgresql://[container name]:5432/tp_db`, while also making sure to be on the same network.

To use environnement variables in a .yml file : 
```yml
datasource:
    url: jdbc:postgresql://${DB_SERVER}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
```

## HTTP Server
### Choose an appropriate base image.
We use `httpd:2.4`.
```
$ tree .
.
├── Dockerfile
└── public-html
    └── index.html
$ docker build -t yuiko911/my-apache2 .
$ docker run -d --net=app-network --name web-serv -p 8081:80 yuiko911/my-apache2
```

And the site is correctly running on localhost:8081

```
$ docker stats web-serv
CONTAINER ID   NAME       CPU %     MEM USAGE / LIMIT     MEM %     NET I/O           BLOCK I/O    PIDS 
2a70afeb26b7   web-serv   0.00%     6.836MiB / 15.32GiB   0.04%     5.28kB / 2.06kB   0B / 4.1kB   82 
```
```
$ docker inspect web-serv
# A lot of information
```
```
$ docker logs web-serv
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.18.0.4. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 172.18.0.4. Set the 'ServerName' directive globally to suppress this message
[Wed Oct 22 08:22:19.941632 2025] [mpm_event:notice] [pid 1:tid 1] AH00489: Apache/2.4.65 (Unix) configured -- resuming normal operations
[Wed Oct 22 08:22:19.941718 2025] [core:notice] [pid 1:tid 1] AH00094: Command line: 'httpd -D FOREGROUND'
172.18.0.1 - - [22/Oct/2025:08:22:23 +0000] "GET / HTTP/1.1" 200 233
172.18.0.1 - - [22/Oct/2025:08:22:23 +0000] "GET /favicon.ico HTTP/1.1" 404 196
172.18.0.1 - - [22/Oct/2025:08:28:49 +0000] "GET / HTTP/1.1" 304 -
```

### Configuration
To get the default configuration, we use the following command, it will dump the config in a `default-config.conf` file 
```
$ docker exec web-serv cat /usr/local/apache2/conf/httpd.conf > default-config.conf
```
Then, to use it in our image, we add this line to the Dockerfile.
```dockerfile
COPY ./my-conf.conf /usr/local/apache2/conf/httpd.conf
```

### Reverse proxy

### Question 1.5
**Why do we need a reverse proxy?**

Reverse proxies are an intermediary between a user's request and the servers. They lets us access different services on different ports through one address 

## Link application

### Docker-compose

### Question 1.6
**Why is docker-compose so important?**

Docker-compose is used to run and manage multiple containers at once, and lets us configure everything at once in one file, which saves a lot of time. 

### Question 1.7
**Document docker-compose most important commands.**

`docker compose up` starts every containers, `-d` is also possible here to start in detached mode
`docker compose down` stops everything
`docker compose build --no-cache` force a rebuild of every containers

### Question 1.8
**Document your docker-compose file.**
(See file)

## Publish

### Question 1.9
**Document your publication commands and published images in dockerhub.**

I didn't do it.

### Question 1.10
**Why do we put our images into an online repo?**

To let other people use them.