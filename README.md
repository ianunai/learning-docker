# Learning Docker

## Getting Started

Tutorial available [here](https://docs.docker.com/get-started/).

### Building an image

Steps to building an image include:

1. Creating a Dockerfile with the appropriate image build instructions.
2. Running the following command:  
`docker build -t getting-started .`  
The `-t` flag denotes the tag to be used for the image being built. The `.` denotes the contents of the current working directory to be used in building the container.

### Running an image

Steps to running an image are included below.

1. Run the following command:  
`docker run -d -p 3000:3000 getting-started`

2. The `-d` flag stands for detached, allowing the container to run in the background.
3. The `-p` flag stands for port mapping, i.e., it instructs docker to map port 3000 of the container to port 3000 of the host machine.
4. The flags can be combined into one, as `-dp`.

### Removing a running container

#### Using CLI

1. Get the ID of the container using `docker ps` command.
2. Stop the container first:  
`docker stop <container id>`
3. Remove the container after having stopped it:  
`docker rm <container id>`

**Note:** The steps 2 and 3 can be combined using the following: `docker rm -f <container id>`.

## Sharing an image

The default Docker registry is Docker Hub. 

### Prerequisites 

1. Ensure that a repository exists on Docker Hub.
2. User is logged in to Docker Hub through terminal. Use the following command to login:  
`docker login -u <username>`.  
When prompted, enter the password.

### Tagging the image

Run the following command to tag the Docker image built locally:  
`docker tag <local tag> <username>/<repository>`.

### Pushing the image to the registry

Use the following command, after performing a tag to push the image to the registry:  
`docker push <username>/<repository>`.  
If no tag is specified, Docker reverts to using `latest` for the image tag.

## Persisting data

Data can be persisted across multiple container runs for a given image using volumes.

### Using named volume

#### Steps

1. Create a volume in Docker called `todo-db` using  
`docker volume create todo-db`.

2. Run the container again, with the `-v` flag to include volume mount:  
`docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started`.  
We use the named volume `todo-db` and mount it to `/etc/todos` which captures all files created at the path.

#### Inspecting the volume

Running `docker volume inspect todo-db` returns details about the given volume. The `Mountpoint` is the actual location on the disk where the data is stored.

Example:
```json
[
    {
        "CreatedAt": "2021-03-24T19:37:33Z",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/todo-db/_data",
        "Name": "todo-db",
        "Options": {},
        "Scope": "local"
    }
]
```

While running in Docker Desktop, the Docker commands are actually running inside a small VM on the machine. If we wanted to look at the actual contents of the Mountpoint directory, we would need to first get inside of the VM.

### Using bind mounts

Bind mounts, in comparison to `Volume`, give the end user control over the exact mountpoint on the host.

#### How To

The example uses a container running in dev-mode, i.e., a container supporting a development workflow.

Start by running the following command:  
```shell script
docker run -dp 3000:3000 \
    -w /app -v "$(pwd):/app" \
    node:12-alpine \
    sh -c "yarn install && yarn run dev"
``` 

1. `-dp 3000:3000` - The `-d` flag allows the container to run in detached mode. The `-p` flag maps the port 3000 of the container to port 3000 of the host.
2. `-w /app` - Sets the "working directory", or the current directory that the command will run from.
3. `-v "$(pwd):/app"` - bind mount the current directory from the host in the container into the `/app` directory on the container.
4. `node:12-alpine` - base image to use
5. `sh -c "yarn install && yarn run dev"` - install all dependencies, including dev (since the `--production` flag from Dockerfile is not included) and run the Node.js app using `yarn run dev`.

Using bind mounts is very common for local development setups. The advantage is that the dev machine doesn't need to have all the build tools and environments installed.

### Multi-Container Apps

Remember this - in general, each container should do one thing and do it well.  

Possible reasons:

* Front-ends and APIs might need scaling differently than databases in the future
* Separate containers allow version control in isolation
* Allows ease in transition to production, where the database may be a managed service
* Running multiple processes on a container requires a process manager, thereby increasing the complexity of container start-up and shut-down  

#### Container Networking

Remember this - **If two containers are on the same network, they can talk to each other. If they aren't, they can't.**


#### Starting a mySQL container

1.We will use a network in Docker and attach the running containers to the network:  
`docker network create todo-app`

2.Starting a mySQL container and attaching it to the network:
```shell script
docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:5.7
```
**Notes:**  
  * We are mapping a volume `todo-mysql-data` that we haven't created, however Docker recognizes that the user needs a named volume and creates one automatically.
  * The `-e` fkag stands for environment variables, mySQL container requires some environment varaibles, mainly the root user password as well as the database name.  

3.To verify whether the container is running, connect using the following command:  
`docker exec -it <mysql-container-id> mysql -p`

Viewing the databases should display the `todos` database as follows:

```
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.00 sec)
```

#### Connecting to the mySQL container

We will make use of the `nicolaka/netshoot` container which ships with various tools to assist with troubleshooting or debugging networking issues.

1.Start by running the `netshoot` container:  
`docker run -it --network todo-app nicolaka/netshoot`
2.Inside the container, we're going to use the `dig` command, which is a useful DNS tool. We're going to look up the IP address for the hostname `mysql`.

```shell script
 $ dig mysql

; <<>> DiG 9.16.11 <<>> mysql
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16175
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;mysql.				IN	A

;; ANSWER SECTION:
mysql.			600	IN	A	172.18.0.2

;; Query time: 3 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Thu Mar 25 16:51:13 UTC 2021
;; MSG SIZE  rcvd: 44
```

Although `mysql` is not a valid hostname, `dig` resolves the `--network-alias mysql` to an IP address found in the `ANSWER SECTION:`.
The advantage of using `--network-alias` is that containers in a network can refer to the alias in order to connect to that container.

#### Running the app with mySQL container

Run the following command:  
```shell script
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:12-alpine \
  sh -c "yarn install && yarn run dev"
```

**Note:**

* `--network todo-app` tells the container to use the `todo-app` network created previously, on which the mySQL container is running.
* `-e` flag is used to specify environment variables by the container, followed by the environment variable's key-value pair.

In the next section, we focus on using Docker Compose to manage multiple deployments in Docker more conveniently and efficiently.

### Using Docker Compose

Docker Compose is a tool that was developed to help define and share multi-container applications. 
With Compose, we can create a YAML file to define the services and with a single command, can spin everything up or tear it all down.

#### Installing Docker Compose

Available with Docker Desktop, can run the following command to verify:  
`$ docker-compose version`

#### Creating a Docker Compose File

Creating a `docker-compose.yaml` at the project root and filling it resulting in:

```yaml
version: "3.7"

services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:5.7
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

**Notes:**
1. The `version` is for the schema version for Docker Compose, it's advised to go with the latest version.
2. `services` are used to define the services in the application.
3. The name for a service, such as `app` or `mysql` also ends up being used by Docker as a network alias.
4. Docker compose can detect bind mounts and allows relative paths, however, it does not create a named volume unless it is defined, unlike docker CLI. Hence, we explicitly define it under `volumes`.

#### Running the Application Stack

1. Application stack can be run using the `docker-compose up command`:  
`docker-compose up -d` (the `-d` flag ensures it runs in the background).  
By default, `docker-compose` creates a network for the application stack.

2. Logs can be viewed using:  
`docker-compose logs -f`.  
  In order to view logs for a service, specify the service name, as:  
  `docker-compose logs -f app` or `docker-compose logs -f mysql`
  
3. The containers spawned using `docker-compose` have the following nomenclature:  
`<project-name>_<service-name>_<replica-number>`.

#### Removing the Deployed Stack

Issuing `docker-compose down` tears down the running application stack. By default, the volumes are not deleted when this command is run. In order to remove the volume, we need to include `--volumes` flag to the command:  
`docker-compose down --volumes`.

### Best Practices for Image Building

#### Image Scanning

Image scanning can be done through Docker CLI using [Snyk](http://snyk.io/),  
`docker scan <image name>`.  
Apart from scanning images through Docker CLI, Docker Hub can also be configured to perform image scans.

#### Image Layering

Using `docker image history`, we can view the layers that went in to building an image.

For example,  
```shell script
$ docker image history getting-started
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
6222c93a09ea   3 hours ago   CMD ["node" "src/index.js"]                     0B        buildkit.dockerfile.v0
<missing>      3 hours ago   COPY . . # buildkit                             6.69MB    buildkit.dockerfile.v0
<missing>      3 hours ago   RUN /bin/sh -c yarn install --production # b…   85.2MB    buildkit.dockerfile.v0
<missing>      3 hours ago   COPY package.json yarn.lock ./ # buildkit       180kB     buildkit.dockerfile.v0
<missing>      3 hours ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   /bin/sh -c #(nop)  CMD ["node"]                 0B        
<missing>      2 weeks ago   /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B        
<missing>      2 weeks ago   /bin/sh -c #(nop) COPY file:238737301d473041…   116B      
<missing>      2 weeks ago   /bin/sh -c apk add --no-cache --virtual .bui…   7.62MB    
<missing>      2 weeks ago   /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.5      0B        
<missing>      2 weeks ago   /bin/sh -c addgroup -g 1000 node     && addu…   75.7MB    
<missing>      2 weeks ago   /bin/sh -c #(nop)  ENV NODE_VERSION=12.21.0     0B        
<missing>      4 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B        
<missing>      4 weeks ago   /bin/sh -c #(nop) ADD file:7eeea546ecde7a036…   5.61MB    
```

Adding the `--no-trunc` flag allows us to view the complete output of the above command.

#### Layer Caching

Remember this - **once a layer changes, all downstream layers have to be recreated as well**.

#### Multi-stage Builds

Multi-stage builds are an incredibly powerful tool to help use multiple stages to create an image. There are several advantages to them:

* Separate build-time dependencies from runtime dependencies
* Reduce overall image size by shipping only what your app needs to run

##### Maven/Tomcat Example

When building Java-based applications, a JDK is needed to compile the source code to Java bytecode. However, that JDK isn't needed in production. Also, we might be using tools like Maven or Gradle to help build the app. Those also aren't needed in our final image. Multi-stage builds help.

Example:  
```dockerfile
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps 
```

In this example, we use one stage (called `build`) to perform the actual Java build using Maven. In the second stage (starting at `FROM tomcat`), we copy in files from the `build` stage. The final image is only the last stage being created (which can be overridden using the `--target` flag).

### Next Steps

#### Container Orchestration

Using Kubernetes, Docker Swarm, ECS, and so on allows us to run and manage applications more effectively across multiple machines and clusters.

