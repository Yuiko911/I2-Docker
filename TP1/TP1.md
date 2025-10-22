# Part 1
## Database
### Basics
```sh
$ docker build -t yuiko911/postgres
[...]
Successfully built 072c46f43171
Successfully tagged yuiko911/postgres:latest
```

We create a network to isolate our containers from the outside network, and have them be able to connect to each other easily. Then, we start the Postgres container on this network, in detached mode.
```sh
$ docker network create app-network 
$ docker run -d --network app-network yuiko911/postgres:latest 
```

Running Adminer in detached mode.
```sh
$ docker run -d --network app-network -p 8080:8080 adminer 
```
*Note: In the "Server" field of Adminer, we should not put "localhost", but rather the name of the container, in this case "epic_wilbur"*

### Question 1-1
**For which reason is it better to run the container with a flag -e to give the environment variables rather than put them directly in the Dockerfile?** \
Environment variables often contain sensitive information, such as passwords. Putting a password in the Dockerfile is insecure, as it is written directly in easily recognisable plain text, accesible for an attacker to search, find and see it. Using -e, that risk is minimized, as it's only written once on lauch (though the risk is still present if the attacker has acces to the command history) 

We can remove the environment variables from the Dockerfile...
```sh
$ docker rm -f epic_wilbur
$ docker image rm yuiko911/postgres:latest
# Removing the environment variables from the Dockerfile
$ docker build -t yuiko911/postgres .
[...]
Successfully built 2bf60600670f
Successfully tagged yuiko911/postgres:latest
```
...and then run the container again.
```sh
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
```sh
$ docker rm -f postgres_con
$ docker image rm yuiko911/postgres:latest
$ docker build -t yuiko911/postgres .
[...]
Successfully built fe5853b5df8c
Successfully tagged yuiko911/postgres:latest
```

...and restart the services.
```sh
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
```sh
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
```sh
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
```sh
# Building the image
$ docker built -t yuiko911/postgres .

# Running Postgres
$ docker run -d --name=postgres_con --network=app-network -e POSTGRES_DB=tp_db -e POSTGRES_USER=tp_usr -e POSTGRES_PASSWORD=tp_pwd -v /home/yuuka/Documents/EFREI/I2-Docker/TP1/Postgres-data:/var/lib/postgresql/data yuiko911/postgres:latest

# Running Adminer
$ docker run -d --name=adminer --net=app-network -p "8090:8080" adminer
```

## Backend API
### Basics
```sh
# Making sure to use Java 21 to compile
$ javac Main.java
$ docker built -t yuiko911/backendapi .
Successfully built 236ddaad83f5
Successfully tagged yuiko911/backendapi:latest
$ docker run yuiko911/backendapi:latest  
Hello World!
```

```sh
$ docker build -t yuiko911/simpleapi .
$ docker run -d --name=simple_api -p 8080:8080 yuiko911/simpleapi:latest
```
*Note: Make sure to not use the same port as another container (Adminer in my case)*

The URL scheme for the database is `jdbc:postgresql://[container name]:5432/tp_db`, while also making sure to be on the same network.