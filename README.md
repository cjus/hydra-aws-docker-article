# Deploying Node Microservices to AWS using Docker

* Introduction to docker and building containers
* Running containers on AWS
* Building and working with a swarm cluster

---

In this two part post we'll look at building and deploying microservices to Amazon's AWS using Docker. In this first part we'll focus on building a simple microservice and packaging it in a docker container. We'll also step through hosting the container on AWS. In part two, we'll build a cluster on AWS using Docker Swarm mode.

Because doing such a thing is plagued with lots of complexity, we're going to use a microservices library called [Hydra](https://www.npmjs.com/package/hydra)  -  which will greatly simply the effort while offering considerable scalability benefits. Even if you choose not to use Hydra, the information in this post should help you get started with AWS and Docker.

A quick recap if you're wondering what this Hydra thing is. Hydra is a NodeJS package which facilitates building distributed applications such as Microservices. Hydra offers features such as service discovery, distributed messaging, message load balancing, logging, presence, and health monitoring. As you can imagine, those features would be of use for any service living on cloud infrastructure.

You don't need to really know how to use Hydra to get a lot out of this article - but if you're interested then you can learn more by reviewing two earlier posts on Hydra here on RisingStack. The first is [Building ExpressJS-based microservices using Hydra](https://community.risingstack.com/tutorial-building-expressjs-based-microservices-using-hydra/), and the second is [Building a Microservices Example Game with Distributed Messaging](https://community.risingstack.com/building-a-microservices-example-game-with-distributed-messaging/). A microservice game? Seriously? For the record, I do reject claims that I have too much time on my hands. 😃

We'll begin by recapping docker containerization  - just in case you're new to this. Feel free to skim or skip over the next section, if you're already familiar with docker.

### Containerization?

Virtual Machine software has ushered in the age of containerization where applications can be packaged as containers making them portable and easier to manage. Docker is a significant evolution of that trend.

Running microservices inside of containers means we're able to run containers locally on our laptops and run the same containers in the cloud. This greatly simplifies the construction of larger applications that consist of many moving service parts, as you're able to debug locally.

Packaging your ExpressJS applications inside of a Docker container is straightforward.

Download and install the Docker community edition from docker.com 

* `cd` into an existing project folder
* Build a simple service
* Create a Dockerfile (see example below)
* Run: `docker build -t hello-service:0.0.1 .` Don't forget the trailing period which specifies the working directory.

The tag for the command above specifies your service name and version. It's a good practice to prefix that entry with your username or company name. For example: `cjus/hello-service:0.0.1` If you're using Docker hub to store your containers then you'll need definitely need to prefix your container name. We'll touch on Docker hub a bit more later.

### Building a simple microservice

To build our simple microservice we'll use a package called Hydra-express, which creates a microservice using Hydra and ExpressJS. Why not just use ExpressJS? By itself, an ExpressJS app only allows you to build a Node server and add API routes. However, that basic server isn't a complete microservice. In comparison, a Hydra-express app includes functionality to discover other Hydra apps and load balance requests between them using presence and health information. Those capabilities will become important when we consider applications running on AWS and in a Docker Swarm cluster. Building Hydra and Hydra-Express apps is covered in more detail in my earlier [RisingStack articles](https://community.risingstack.com/author/carlos/) on Hydra.

This approach does however, require that you're running a local instance of Redis. In the extremely unlikely event that you're unfamiliar with Redis - checkout this [quick start page](https://www.hydramicroservice.com/docs/quick-start/step1.html).

To avoid manually typing the code for a basic hydra-express app we'll install [Yeoman](http://yeoman.io/learning/) and Eric Adum's excellent [hydra app generator](https://github.com/flywheelsports/generator-fwsp-hydra). A Yeoman generator asks a series of questions and then generates an app for you. You can then customize it. This is similar to running the [ExpressJS Generator](https://expressjs.com/en/starter/generator.html).

```shell
$ sudo npm install -g yo generator-fwsp-hydra
```

Next, we'll invoke Yeoman and the hydra generator.  Name your microservice `hello` and make sure to specify a port address of 8080 - you can then choose defaults for the remaining options.

```shell
$ yo fwsp-hydra
fwsp-hydra generator v0.2.10   yeoman-generator v1.1.1   yo v1.8.5
? Name of the service (`-service` will be appended automatically) hello
? Your full name? Carlos Justiniano
? Your email address? cjus34@gmail.com
? Host the service runs on?
? Port the service runs on? 8080
? What does this service do? Says hello
? Does this service need auth? No
? Is this a hydra-express service? Yes
? Set up a view engine? No
? Set up logging? No
? Enable CORS on serverResponses? No
? Run npm install? No
   create hello-service/specs/test.js
   create hello-service/specs/helpers/chai.js
   create hello-service/.editorconfig
   create hello-service/.eslintrc
   create hello-service/.gitattributes
   create hello-service/.nvmrc
   create hello-service/.gitignore
   create hello-service/package.json
   create hello-service/README.md
   create hello-service/hello-service.js
   create hello-service/config/sample-config.json
   create hello-service/config/config.json
   create hello-service/scripts/docker.js
   create hello-service/routes/hello-v1-routes.js

Done!
'cd hello-service' then 'npm install' and 'npm start'
```

You'll end up with a folder called hello-service.

```shell
$ tree hello-service/
hello-service/
├── README.md
├── config
│   ├── config.json
│   └── sample-config.json
├── hello-service.js
├── package.json
├── routes
│   └── hello-v1-routes.js
├── scripts
│   └── docker.js
└── specs
    ├── helpers
    │   └── chai.js
    └── test.js

5 directories, 9 files
``` 

The assute reader will notice that there's a scripts folder with a `docker.js` file.  More on that later!

After cd-ing into the folder you can can build using `npm install`, and after running `npm start` you should see:

```shell
$ npm start

> hello-service@0.0.1 start /Users/cjus/dev/hello-service
> node hello-service.js

INFO
{ event: 'start',
  message: 'hello-service (v.0.0.1) server listening on port 8080' }
INFO
{ event: 'info', message: 'Using environment: development' }
serviceInfo { serviceName: 'hello-service',
  serviceIP: '192.168.1.151',
  servicePort: 8080 }
```

Using the IP address and Port above we can access our `v1/hello` route from a web browser:

![](./192_168_1_151_8080_v1_hello.png)

Note, I'm using the excellent [JSON Formatter](https://chrome.google.com/webstore/detail/json-formatter/bcjindcccaagfpapjjmafapmmgkkhgoa) chrome extension. Without a similar browser extension you'll just see this:

```
{"statusCode":200,"statusMessage":"OK","statusDescription":"Request succeeded without error","result":{"greeting":"Welcome to Hydra Express!"}}
```

### Creating the Dockerfile

If you're following along and used the hydra generator, you already have a

The docker build step above looks for a file called `Dockerfile` to specify the specifics of your container. The first line specifies the base image that will be used for your container. We specify the light-weight (Alpine) image containing a minimal Linux and NodeJS version 6.9.4  -  however, you can specify the larger standard Linux image just using: FROM: node:6.9.4

```
FROM node:6.9.4-alpine
MAINTAINER Carlos Justiniano cjus34@gmail.com
EXPOSE 8080
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
ADD . /usr/src/app
RUN npm install --production
CMD ["npm", "start"]
```

Other important entry, EXPOSE, is the port that our Express app listens on. The remaining lines specify that the contents of the current directory should be copied to /usr/src/app inside of the container. We then instruct Docker to run the npm install command to pull package dependencies. The final line specifies that npm start will be invoked when the container is executed.

### Running our container

We use the docker run command to invoke our container and service. The `-d` command specifies that we want to run in daemon (background mode) and the `-p` command publishes our services ports. The port syntax says: "on this machine use port 8080 and map that to the containers internal port" which is also 8080. We also name the service using the  `--name` flag  -  that's useful otherwise Docker will provide a random name for our running container. The last portion shown below is the service name and version. Ideally that should match the version in your package.json file.

```
$ docker run -d -p 8080:8080 --name hello-service hello-service:0.0.1
```

### Communicating with our container

At this point you should be able to open your web browser and point it to http://localhost:8080 to access your service.

### Sharing your containers

Now that you've created a container you can share it with others by publishing it to a container repository such as Docker Hub. You can setup a free account  which will allow you to publish unlimited public containers, but you'll only be able to publish one private container. To maintain multiple private containers you'll need a paid subscription. However, the plans start as low as $7 per month.

You, or others, can pull a container image from your docker hub repo using one simple command:

```
$ docker pull cjus/hello-service:0.0.1
```
