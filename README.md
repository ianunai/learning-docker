# Learning Docker

## Getting Started

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

