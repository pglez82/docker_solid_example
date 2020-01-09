# Docker Solid Example
This example will be based in this two git repositories:
* [node-solid-server](https://github.com/solid/node-solid-server)
* [profile-viewer-react](https://github.com/solid/profile-viewer-react): very basic Solid example application. It just shows a the name of a Solid user and its friends. The important thing is that this project uses [solid-react-components](https://github.com/solid/react-components) and we can use it as a template for starting our own Solid application.

The aim of this example is automate the build and deployment of a Solid server and the example profile viewer application using Docker. For this, we will use **docker-compose because** our system has two very different parts (the server and the webapp). Docker-compose allows us to deploy multiple containers easily.

Prerequisites: [Docker basics](https://github.com/pglez82/docker_cheatsheet/).

## Step 1 - The server
The Solid server is hosted in [node-solid-server](https://github.com/solid/node-solid-server). If you browse the repository, you can see that this project already has a Dockerfile, that means it is prepared to be built as a Dockerimage. Lets start cloning the repository:
```
git clone https://github.com/solid/node-solid-server
```
If we check the Docker file we can see that it is based in `8.11.2-onbuild`. This kind of node images **are deprecated**, so we are going to change this file to use a newer noder image:
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
* We generate a new self-signed certificate using `openssl`. This is only valid for development and for production we would have to get a valid certificate.
* We start the server.

If we want to test the server, we just need to run:
```
docker build -t solidserver .
``` 
