# Deploying Node ExpressJS Microservices to AWS using Docker and Docker Swarm Mode

In this post we'll look at deploying microservices into a Docker Swarm. Because doing such a thing is pledged with lots of complexity, we're going to use a microservices library called [Hydra](https://www.npmjs.com/package/hydra) — which will greatly simply the effort while offering considerable scalability benefits. Even if you choose not to use Hydra, the information in this post should help get started with Docker Swarm.
A quick recap if you're wondering what this Hydra thing is. 

Hydra is a NodeJS package which facilitates building distributed applications such as Microservices. Hydra offers features such as service discovery, distributed messaging, message load balancing, logging, presence, and health monitoring. You can learn more at > [What is Hydra](https://www.hydramicroservice.com/what-is-hydra.html).

You can find two earlier posts on Hydra here on RisingStack. The first is [Building ExpressJS-based microservices using Hydra](https://community.risingstack.com/tutorial-building-expressjs-based-microservices-using-hydra/), and the second is [Building a Microservices Example Game with Distributed Messaging](https://community.risingstack.com/building-a-microservices-example-game-with-distributed-messaging/). For the record, I do reject claims that I have too much time on my hands!

I also wrote a longer post called xxxx, which is a dryer and less friendly version of this post. If you find that you need a lot more details, refer to that post. 

We'll begin by recapping docker containerization — in case you're new to this. Feel free to skim or skip over that section if you've been using docker for some time now.

## Containerization?

Virtual Machine software has ushered in the age of containerization where applications can be packaged as containers making them portable and easier to manage. Docker is a significant evolution of that trend.
Running microservices inside of containers means we're able to run containers locally on our laptops and run the same containers in the cloud. This greatly simplifies the construction of larger applications that consist of many moving service parts, as you're able to debug locally.

Packaging your ExpressJS applications inside of a Docker container is straightforward.

Download and install the Docker community edition from docker.com 

* cd into an existing project folder.
* Create a Dockerfile (see example below)
* Run: docker build -t myservice:0.0.1 .

The tag for the command above specifies your service name and version. It's a good practice to prefix that entry with your username or company name. For example: cjus/myservice:0.0.1 If you're using Docker hub to store your containers then you'll need definitely need to prefix your container name. We'll touch on Docker hub a bit more later.
Creating the Dockerfile

The docker build step above looks for a file called Dockerfile to specify the specifics of your container. The first line specifies the base image that will be used for your container. We specify the light-weight (Alpine) image containing a minimal Linux and NodeJS version 6.9.4 — however, you can specify the larger standard Linux image just using: FROM: node:6.9.4

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

## Running our container

We use the docker run command to invoke our container and service. The `-d` command specifies that we want to run in daemon (background mode) and the `-p` command publishes our services ports. The port syntax says: "on this machine use port 8080 and map that to the containers internal port" which is also 8080. We also name the service using the  `—-name` flag — that's useful otherwise Docker will provide a random name for our running container. The last portion shown below is the service name and version. Ideally that should match the version in your package.json file.

```
$ docker run -d -p 8080:8080 --name myservice myservice:0.0.1
```

## Communicating with our container

At this point you should be able to open your web browser and point it to http://localhost:8080 to access your service.

## Sharing your containers
Now that you've created a container you can share it with others by publishing it to a container repository such as Docker Hub. You can setup a free account — but that will only allow you to publish public container. To maintain private containers you'll need a
