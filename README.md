# Docker Solid Example
This example will be based in this two git repositories:
* [node-solid-server](https://github.com/solid/node-solid-server)
* [profile-viewer-react](https://github.com/solid/profile-viewer-react): very basic Solid example application. It just shows a the name of a Solid user and its friends. The important thing is that this project uses [solid-react-components](https://github.com/solid/react-components) and we can use it as a template for starting our own Solid application.

The aim of this example is automate the build and deployment of a Solid server and the example profile viewer application using Docker. For this, we will use **docker-compose** because our system has two very different parts (the server and the webapp). Docker-compose allows us to deploy multiple containers easily.

Prerequisites: Understand the [Docker basics](https://github.com/pglez82/docker_cheatsheet/).

## Step 1 - The server
The Solid server is hosted in [node-solid-server](https://github.com/solid/node-solid-server). If you browse the repository, you can see that this project already has a Dockerfile, that means it is prepared to be built as a Dockerimage. Lets start cloning the repository:
```
git clone https://github.com/solid/node-solid-server
```
If we check the Docker file we can see that it is based in `8.11.2-onbuild`. This kind of node images **are deprecated**, so we are going to change this file to use a newer node image:
#### **`node-solid-server/Dockerfile`**
```
FROM node:12.14.1
EXPOSE 8443
COPY . /usr/src/app
WORKDIR /usr/src/app
RUN npm install
COPY config.json-default config.json
RUN openssl req \
    -new \
    -newkey rsa:4096 \
    -days 365 \
    -nodes \
    -x509 \
    -subj "/C=US/ST=Denial/L=Springfield/O=Dis/CN=www.example.com" \
    -keyout privkey.pem \
    -out fullchain.pem
CMD npm run solid start
```
Lets explain this docker file:
* The image will be based on node:12.14.1. 
* We copy the application will be copied from local to the `/usr/src/app` dir in the image.
* We install the dependencies defined in `packages.json` using `npm install`.
* We copy the configuration file `config.json` (here we are using the default configuration but obviously this could be changed).
* We generate a new self-signed certificate using `openssl`. This is only valid for development and for production we would have to get a valid certificate (see details [here](https://github.com/solid/node-solid-server).
* We start the server.

Note: This Dockerfile has more lines than the original. The reason is that the `8.11.2-onbuild` image used in the original Dockerfile was designed to automatically pick the source code from the directory where the Dockerfile is. We are using a normal node image so we have to do this steps by hand.

If we want to test the server, we just need to run:
```
docker build -t solidserver .
docker run --name solidserver -p 8443:8443 solidserver
``` 
If we go to https://localhost:8443 we can access our pod server. 

Note: If we create a pod in our server we should follow the instructions in [node-solid-server](https://github.com/solid/node-solid-server) in the section `Run multi-user server (intermediate)` in order to be able to access youruser.localhost.

## Step 2 - The webapp
In this step we will prepare our sample webapp. Lets first clone the repository:
```
git clone https://github.com/solid/node-solid-server
```
Note: be careful not to clone this project inside the server. The directory structure should be:

| projectdir
|-- node-solid-server
|-- profile-viewer-react

Lets create a Dockerfile for this project:
#### **`profile-viewer-react/Dockerfile`**
```
FROM node:12.14.1
COPY . /app
WORKDIR /app
RUN npm install
CMD ["npm", "start"]
```
So this Dockerfile is very simple. It justs uses the same base image as before (an image with node installed), we copy the app to the /app directory in the container, install the dependencies and start the server.

If we want to start the webapp, we just need to run:
```
docker build -t solidwebapp .
docker run --name solidwebapp -p 3000:3000 solidwebapp
``` 
If we go to https://localhost:3000 we can access our sample profile viewer. 

## Step 3 - Integrating both parts
Now we have two separate projects, the server and the webapp. We also have two docker files to build two images. Lets use **docker-compose** to automate the process of building and running this two parts. Create a new file docker-compose.yml in your project root directory:

#### **`docker-compose.yml`**
```
version: '3'
services:
  solidserver:
    build: ./node-solid-server/
    ports:
    - "8443:8443"
  sampleweb:
    build: ./profile-viewer-react/
    ports:
    - "3000:3000"
```
This file is very easy two understand. We define that our application has two services, the server and the webapp. We just need to tell docker where are the two parts and which port each part uses. Docker will go to each directory and look for a Dockerfile. It will then build each image and launch them. The final result will be a pod server and a profile viewer running as two docker instances using a fully automated process. The final step will be executing this docker-compose file:
```
docker-compose up
```
