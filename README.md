# Docker Solid Example
This example will be based in this two git repositories:
* [node-solid-server](https://github.com/solid/node-solid-server)
* [profile-viewer-react](https://github.com/solid/profile-viewer-react): very basic Solid example application. It just shows a the name of a Solid user and its friends. The important thing is that this project uses [solid-react-components](https://github.com/solid/react-components) and we can use it as a template for starting our own Solid application.

The aim of this example is automate the build and deployment of a Solid server and the example profile viewer application using Docker. For this, we will use **docker-compose** because our system has two very different parts (the server and the webapp). Docker-compose allows us to deploy multiple containers easily.

Prerequisites: Understand the [Docker basics](https://github.com/pglez82/docker_cheatsheet/).

## Step 1 - The server
The Solid server is hosted in [node-solid-server](https://github.com/solid/node-solid-server). If you browse the repository, you can see that this project already has a Dockerfile, that means it is prepared to be built as a Dockerimage. Lets start cloning the repository:
```bash
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
* We generate a new self-signed certificate using `openssl`. This is only valid for development and for production we would have to get a valid certificate (see details [here](https://github.com/solid/node-solid-server)).
* We start the server.

Note: This Dockerfile has more lines than the original. The reason is that the `8.11.2-onbuild` image used in the original Dockerfile was designed to automatically pick the source code from the directory where the Dockerfile is. We are using a normal node image so we have to do this steps by hand.

If we want to test the server, we just need to run:
```sh
docker build -t solidserver .
docker run --name solidserver --rm -d -p 8443:8443 solidserver
``` 
Note: `--rm` flag means that the container will be removed after stoping it and `-d` flag allow us to execute the container as a daemon (non iteractive)
If we go to https://localhost:8443 we can access our pod server. 

Note: If we create a pod in our server we should follow the instructions in [node-solid-server](https://github.com/solid/node-solid-server) in the section `Run multi-user server (intermediate)` in order to be able to access youruser.localhost.
You may need to configure your `/etc/hosts` file adding this line to:
```
127.0.0.1	*.localhost
```
In Windows, the `/etc/hosts` file is usually located at: `c:\Windows\System32\Drivers\etc\hosts`

The problem here is after stoping the server with `docker stop solidserver`, all the data that could be stored in the pod server will be lost. Maybe this is what we want (in case we are doing tests), or maybe we want to make sure that the pods data is stored outside the container. For this we will use **volumes** in step 3.

## Step 2 - The webapp
In this step we will prepare our sample webapp. Lets first clone the repository:
```
git clone https://github.com/solid/profile-viewer-react
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

Sometimes is also important to have a `.dockerignore` file. This file is useful for ignoring some files or directories when building a docker image. Lets say that we build our application locally. For that we will use `npm install` and `npm run build`, which will create two directories: `node_modules` and `build`. We do not want these two directories to be used when building the image so we can include them in this file:
#### **`profile-viewer-react/.dockerignore`**
```
node_modules
build
```

Now, if we want to start the webapp, we just need to run:
```
docker build -t solidwebapp .
docker run --name solidwebapp -p 3000:3000 solidwebapp
``` 
If we go to https://localhost:3000 we can access our sample profile viewer. 

As you may notice here, I am using `npm start` in order to launch the application. Obviously this is good for development but not suitable for production. Here you have another `Dockerfile` example that will build the react application and deploy using a better http server:
#### **`profile-viewer-react/Dockerfile`**
```
FROM node:12.14.1
COPY . /app
WORKDIR /app
RUN npm install
RUN npm run build
RUN npm install -g http-server
CMD http-server build -p 3000
```

## Step 3 - Integrating both parts
Now we have two separate projects, the server and the webapp. We also have two docker files to build two images. Lets use **docker-compose** to automate the process of building and running this two parts. Create a new file docker-compose.yml in your project root directory:

#### **`docker-compose.yml`**
```
version: '3'
services:
  solidserver:
    build: ./node-solid-server/
    volumes:
      - ./volumes/soliddata:/usr/src/app/data
      - ./volumes/soliddb:/usr/src/app/.db
    ports:
      - "8443:8443"
  sampleweb:
    build: ./profile-viewer-react/
    ports:
      - "3000:3000"
volumes:
  soliddata:
    external: false
  soliddb:
    external: false
```

This file is very easy two understand. We define that our application has two services, the server and the webapp. We just need to tell docker where are the two parts and which port each part uses. Docker will go to each directory and look for a Dockerfile. It will then build each image and launch them. The final result will be a pod server and a profile viewer running as two docker instances using a fully automated process. The final step will be executing this docker-compose file:
```
docker-compose up -d
```
In order to stop both containers we can use:
```
docker-compose down
```

Also, appart from the services we have defined a volume (with name soliddata). This is were the users pods will be stored. We map the host directory `volumes/soliddata` (relative to `docker-compose.yml`) with the container directory `/usr/src/app/data` which is the dir where solid stores the profiles. We also do the same with the passwords that are stored in the `.db` folder using a second volume. This way we ensure that even after removing the containers (which happens when we execute `docker-compose down`) our data will persist, and if we start them up again, the pods data will be available.

Also, if we change something in any of the projects (for instance, a change in the profile viewer), we need to recreate the images. In order to force `docker-compose` to recreate the images before launching the containers, it can be executed like this:
```
docker-compose up --force-recreate --build
```
## Load tests (Gatling)
In order to use Gatling for doing the load tests in our application we need to [download](https://gatling.io/open-source/start-testing/) it. Basically, the program has two parts, a [recorder](https://gatling.io/docs/current/http/recorder) to capture the actions that we want to test and a program to run this actions and get the results. Gatling will take care of capture all the response times in our requests and presenting them in quite useful graphics for its posterior analysis.

Once we have downloaded Gatling we need to start the recorder. This works as a proxy that intercepts all the actions that we make in our browser. That means that we have to configure our browser to use a proxy.

![Gatling proxy](images/gatling1.png?raw=true "Gatling 1")

In this case the proxy will work in the port 8000. Now we need to tell Firefox that we want to use this proxy. Here is important to note that Firefox by deffault will not use a proxy if the address is localhost. In order to do this, we need to set the property `network.proxy.allow_hijacking_localhost` to `true` in `about:config`. 

**Important note**: We are setting this example having the application in the same machine than Gatling. This is not a good practice as Gatling generates overhead in the machine that affect the tests. One good way of doing the test is using a service like Amazon AWS or Google Cloud. This way we can deploy the application there using docker and launch the Gatling load tests from our local machine. Another advantage of this system is that we will be able to test different server machines (increase RAM, number of cores, etc) until we reach the performance that we need for our application. Also, depending on the type of application it will be possible to deploy the application using multiple containers and use a load balancer, so our application will be more scalable. 

Once we have the recorder configured, and the application running (server and webapp), we can start recording our first test. We must specify a package and class name. This is just for test organization. Package will be a folder and Class name the name of the test. In my case I have used `profileviewer` and `LoadTestLoginExample`. I have also pressed the button `No static resources` so the file won't get to complex with two many petitions. After pressing start the recorder will start capturing our actions in the browser. So here you should perform all the the actions that you want to record. In this example we will be recording the login process. Here is the resulting file, in [Scala](https://www.scala-lang.org/).

**Important note**: This test assume that we have created the acccount testaccount/Testaccount_123 in our solid server. Also that we have configured the `hosts` file so this profile is accesibile.

#### **`gatlingdir/user-files/simulations/profileviewer/LoadTestLoginExample.scala`**
```scala
package profileviewer

import scala.concurrent.duration._

import io.gatling.core.Predef._
import io.gatling.http.Predef._
import io.gatling.jdbc.Predef._

class LoadTestLoginExample extends Simulation {

	val httpProtocol = http
		.baseUrl("https://localhost:8443")
		.inferHtmlResources(BlackList(""".*\.js""", """.*\.css""", """.*\.gif""", """.*\.jpeg""", """.*\.jpg""", """.*\.ico""", """.*\.woff""", """.*\.woff2""", """.*\.(t|o)tf""", """.*\.png""", """.*detectportal\.firefox\.com.*"""), WhiteList())
		.acceptHeader("text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8")
		.acceptEncodingHeader("gzip, deflate")
		.acceptLanguageHeader("en-US,en;q=0.5")
		.doNotTrackHeader("1")
		.userAgentHeader("Mozilla/5.0 (X11; Linux x86_64; rv:72.0) Gecko/20100101 Firefox/72.0")

	val headers_0 = Map(
		"Accept-Encoding" -> "gzip, deflate",
		"Upgrade-Insecure-Requests" -> "1")

	val headers_2 = Map(
		"Accept" -> "*/*",
		"Origin" -> "http://localhost:3000")

	val headers_4 = Map(
		"Accept" -> "*/*",
		"Content-Type" -> "application/json",
		"Origin" -> "http://localhost:3000")

	val headers_5 = Map("Upgrade-Insecure-Requests" -> "1")

	val headers_6 = Map(
		"Origin" -> "https://localhost:8443",
		"Upgrade-Insecure-Requests" -> "1")

    val uri1 = "localhost"

	val scn = scenario("LoadTestLoginExample")
		.exec(http("request_0")
			.get("http://" + uri1 + ":3000/")
			.headers(headers_0))
		.pause(1,5)
		.exec(http("request_1")
			.get("http://" + uri1 + ":3000/popup.html")
			.headers(headers_0))
		.pause(1,5)
		.exec(http("request_2")
			.get("/.well-known/openid-configuration")
			.headers(headers_2)
			.resources(http("request_3")
			.get("/jwks")
			.headers(headers_2),
            http("request_4")
			.post("/register")
			.headers(headers_4)
			.body(RawFileBody("profileviewer/loadtestloginexample/0004_request.json")),
            http("request_5")
			.get("/authorize?scope=openid&client_id=19442505d23842d599e13d84a8f2e01e&response_type=id_token%20token&request=eyJhbGciOiJub25lIn0.eyJyZWRpcmVjdF91cmkiOiJodHRwOi8vbG9jYWxob3N0OjMwMDAvcG9wdXAuaHRtbCIsImRpc3BsYXkiOiJwYWdlIiwibm9uY2UiOiJVUVdmcVZ2alI4ZDhGZ0h5V2RyakJ4cGlkUWFTdWNtd1FPd1RZRlAtV25FIiwia2V5Ijp7ImFsZyI6IlJTMjU2IiwiZSI6IkFRQUIiLCJleHQiOnRydWUsImtleV9vcHMiOlsidmVyaWZ5Il0sImt0eSI6IlJTQSIsIm4iOiJ1MHU5amNTQVlpa3Rtd3RXUGp6YXViajV1WE5zcTJmbmxUTVVpQzB5YW5pZERybW1LZ0lQeXd2a0tfWUZ3RmVmWm9zN052M0wxZEdGekMtamRFelloWDN5NnZMa2otYVoxSUV4bTRRN0hET2c2MC1xZEJVa0d1bXJOM1UyZmFJcko1dWEySEROOWZGN2dIek5fQ0g2UXR2ZWJuSHQ3RF82cVVhcWJWNEJSRGtRWTJDbWdyX2otMzh3LUZfZ0M2dThCdDA2VG14NkUxNzlVem1vTVNJa0RlQzhGN2dZbDd3Y19nMFNjZEZTZWptNHhSdnZmOTdQZmR5WEYyRFp1S29jMlRpRGtYLWsxN040VS1xQ0tzd216RTFBdFJTdlVwRjdTd3IwY3hhdUZUbmNWYjVWbnRfS3lxbWMzV3Z4ejNwQ1hvbHA2UndseXVBUFczdGRmZUdWWHcifX0.&state=-jlhEX87x67O8V6C3Fts4RN89uFLxHR9aasImLFFTvs")
			.headers(headers_5)))
		.pause(1,5)
		.exec(http("request_6")
			.post("/login/password")
			.headers(headers_6)
			.formParam("username", "testaccount")
			.formParam("password", "Testaccount_123")
			.formParam("response_type", "id_token token")
			.formParam("display", "")
			.formParam("scope", "openid")
			.formParam("client_id", "19442505d23842d599e13d84a8f2e01e")
			.formParam("redirect_uri", "http://localhost:3000/popup.html")
			.formParam("state", "-jlhEX87x67O8V6C3Fts4RN89uFLxHR9aasImLFFTvs")
			.formParam("nonce", "")
			.formParam("request", "eyJhbGciOiJub25lIn0.eyJyZWRpcmVjdF91cmkiOiJodHRwOi8vbG9jYWxob3N0OjMwMDAvcG9wdXAuaHRtbCIsImRpc3BsYXkiOiJwYWdlIiwibm9uY2UiOiJVUVdmcVZ2alI4ZDhGZ0h5V2RyakJ4cGlkUWFTdWNtd1FPd1RZRlAtV25FIiwia2V5Ijp7ImFsZyI6IlJTMjU2IiwiZSI6IkFRQUIiLCJleHQiOnRydWUsImtleV9vcHMiOlsidmVyaWZ5Il0sImt0eSI6IlJTQSIsIm4iOiJ1MHU5amNTQVlpa3Rtd3RXUGp6YXViajV1WE5zcTJmbmxUTVVpQzB5YW5pZERybW1LZ0lQeXd2a0tfWUZ3RmVmWm9zN052M0wxZEdGekMtamRFelloWDN5NnZMa2otYVoxSUV4bTRRN0hET2c2MC1xZEJVa0d1bXJOM1UyZmFJcko1dWEySEROOWZGN2dIek5fQ0g2UXR2ZWJuSHQ3RF82cVVhcWJWNEJSRGtRWTJDbWdyX2otMzh3LUZfZ0M2dThCdDA2VG14NkUxNzlVem1vTVNJa0RlQzhGN2dZbDd3Y19nMFNjZEZTZWptNHhSdnZmOTdQZmR5WEYyRFp1S29jMlRpRGtYLWsxN040VS1xQ0tzd216RTFBdFJTdlVwRjdTd3IwY3hhdUZUbmNWYjVWbnRfS3lxbWMzV3Z4ejNwQ1hvbHA2UndseXVBUFczdGRmZUdWWHcifX0.")
			.check(status.is(302)))

	setUp(scn.inject(atOnceUsers(20))).protocols(httpProtocol)
}
```
The only two things that I have touched in this file is the `pause` function calls. I have set them to be an interval between 1 an 5 seconds, and also, the `atOnceUsers` function parameter. This is a critical parameter that we will have to adjust depending our requirements.

Now that we have a test, we can execute it:

```bash
./gatling.sh -s profileviewer.LoadTestLoginExample
```
The results will be then stored in the results folder. For instance for our 20 users simulation:

![Gatling with 20 users](images/gatling20.png?raw=true "Gatling 20 users")

and if we change the simulation to 50 users:

![Gatling with 50 users](images/gatling50.png?raw=true "Gatling 50 users")

we can see how the response times start to degrade. Obviously the results page provides us with much more information that we can analyze.

### Using docker to execute the load tests
Here is another option to execute the tests. We are going to use docker-compose to start the server and the web, and then launch the tests. Here is how it will be done, starting with the previous docker-compose file.

#### **`docker-compose.yml`**
```
version: '3'
services:
  solidserver:
    build: ./node-solid-server/
    volumes:
      - ./volumes/soliddata:/usr/src/app/data
      - ./volumes/soliddb:/usr/src/app/.db
    ports:
      - "8443:8443"
  sampleweb:
    build: ./profile-viewer-react/
    ports:
      - "3000:3000"
  gatling:
    image: denvazh/gatling
    command: -s profileviewer.LoadTestLoginExample
    depends_on:
      - solidserver
      - sampleweb
    volumes:
      - absolutepathtogatling/conf:/opt/gatling/conf
      - absolutepathtogatling/user-files:/opt/gatling/user-files
      - absolutepathtogatling/results:/opt/gatling/results
    network_mode: "host"
volumes:
  soliddata:
    external: false
  soliddb:
    external: false
```
A few notes about this file:
- We have defined a third service that depends on the other two (we want to wait that the solid server and the website are launched before starting the test.
- This service is based on the `denvazh/gatling`. It is not updated to the last gatling version but it works. We always have the option of building our own gatling image based on his `Dockerfile` if we need to.
- With command we can pass parameters to `gatling.sh`, so we can decide which tests to execute.
- The `network_mode` should be established to `host` for this service. That means that this container will be able to access the host network, thus, it will be able to access http://localhost:3000 and https://localhost:8443.

Finally, in order to run the tests we only need to run:
`docker-compose up`
