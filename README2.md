# Deploying Node Microservices to AWS using Docker (part two)

In [part one](https://community.risingstack.com/deploying-node-js-microservices-to-aws-using-docker/) of this series, we looked at creating a simple microservice and packaging it into a Docker container. We also deployed the container to AWS using Amazon's ECS optimized Linux AMI - which has the Docker engine pre-installed.

In this post, we'll create a Docker Swarm cluster almost entirely from the command line! In the process, we'll deploy multiple services and introduce application and message-based load balancing. We'll continue using [Hydra](https://www.npmjs.com/package/hydra) because it has the singular goal of making microservices approachable!

The architecture outlined in this article will be quite scalable - unless of course you're Netflix and have Netflix size problems. In any case, the approach we'll look at here can be further scaled in complexity to accommodate your specific needs.

Let's get started.

## We'll begin with the end in mind

Our end-goal is to build an eight-node cluster accessible via an Amazon Application Load Balancer (ALB). Our cluster will accept HTTP traffic and load balance between three master nodes which host our service-aware Application API Gateway, [HydraRouter](https://github.com/flywheelsports/hydra-router). HydraRouter, itself a microservice, will be the only service listening on port 80.  It's responsible for routing service calls to individual services within the cluster.

Hydra-router will only run on master nodes 01 - 03, which are accessible via the ALB.  Our microservices will run on worker nodes 01-05.  Services running on worker nodes will not publish ports for use outside of the network that the container is running in.

<img src="./swarm-roles.png" alt="swarm roles" width="600px" />

Referring to the above diagram, the master nodes in the Ingress network communicate with one another in support of high availability. If one master node dies, another is elected the active master. We can also scale the cluster adding and removing machines as required.  

Each Hydra-router running inside of a master node can communicate with microservices running in containers on the service network. Additionally, each service can communicate with the outside world (external API services) and with its internal peers.

Using [Docker swarm mode](https://docs.docker.com/engine/swarm/#feature-highlights), we'll be able to deploy and scale our services using simple commands. When adding and removing EC2 instances participating in a swarm, Docker will redistribute our services across the cluster.

Docker is certainly impressive!

### AWS setup

We're going to use Amazon Web Services. As in the first part of this series, I have to assume that you're somewhat familiar with AWS. You should be comfortable creating EC2 instances and connecting to them using SSH.

Our initial goal with AWS will be to launch machine instances from the command line. In preparation for this, we'll first create a new [IAM role](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create.html) for a programmatic user with `AmazonEC2FullAccess` credentials.

> <img src="./IAM_Management_Console.png" alt="swarm roles" width="600px" />

Make sure to grab the Access Key and Secret Key as you'll need those shortly.

To assist with the creation and configuration of EC2 instances, we'll create a shell script called `create-node` which uses the [docker-machine](https://docs.docker.com/machine/overview/) command to create an EC2 instance and install the Docker engine.

```shell
#!/bin/bash

AWS_AK="FAKE-AKIAISQOOR3SGN3RPCBT"
AWS_SK="FAKE-CfZ77GEyomrp7+VmRCX+yq+D6uJoKv8lufbEH8Jl"
AWS_VPC='vpc-{here}'
AWS_REGION='us-east-1'

NODE_NAME=$1
MTYPE=$2
MACHINE_TYPE=${MTYPE:='t2.small'}

docker-machine create --driver amazonec2 \
    --amazonec2-access-key $AWS_AK \
    --amazonec2-secret-key $AWS_SK \
    --amazonec2-vpc-id "${AWS_VPC}" \
    --amazonec2-region "${AWS_REGION}" \
    --amazonec2-instance-type "${MACHINE_TYPE}" \
    ${NODE_NAME}

echo "${NODE_NAME} should be available in a minute."
```

In this script, we've defined the AWS Access token key `AWS_AK` and the Secret token key `AWS_SK`. Replace the fake values shown with the access key and secret key you copied earlier. Additionally, we define the AWS VPC id `AWS_VPC` and the AWS Region `AWS_REGION`.  Provide values which reflect your Amazon setup. As a best practice, use environment variables to define and export those tokens outside of the script. They're shown here for clarity.

The above script also allows you to specify the type of EC2 instance to use. The default is `t2.small` but could be `t2.micro` or larger depending on your needs.

Using the script is as easy as:

```shell
$ ./create-node node01 t2.micro
```

As a complement to the above script, we'll also create a `remove_node` script.

```shell
#!/bin/bash
docker-machine rm -f $1
```

So we can remove EC2 instances created using `remove-node`:

```shell
$ ./remove_node node01
```

If you haven't created EC2 instances in this way, then those two scripts will be great takeaways. Read on; there's a whole lot more in store!

### Creating EC2 Nodes

As a recap here is the breakdown of the EC2 instances, we'll create.

<img src="./swarm.png" alt="swarm" width="500px" />

Using our `create-node` script we're able to automate the creation and configuration of our EC2 instances.

```shell
# create master nodes
#
for i in 1 2 3
do
  ./create-node master0$i t2.small
done

# create worker nodes
#
for i in 1 2 3 4 5
do
  ./create-node worker0$i t2.small
done
```
> 🍺 **PubTip**: Consider running each section above in different terminal shells. At this stage, the master and worker nodes don't depend on one another, so you can create them in parallel.

Once the above commands complete, we can view a list of machines.

```shell
$ docker-machine ls -t "30"
```

### AWS security group setup

After creating your first EC2 node above you should see a `docker-machine` security group in the VPC you specified. It's a basic setup suitable for simple uses, but we'll need to update it for use with our swarm.

Here's a summary of the changes we need to make:

  * SSH port 22
  * TCP port 2377 for cluster management communications
  * TCP and UDP port 7946 for communication among nodes
  * TCP and UDP port 4789 for overlay network traffic

Your enhanced security group should include the following.

> <img src="./EC2_DockerSecurityGroup.png" alt="AWS security group" />

With these changes in place, we can proceed to configure our swarm.

## Redis setup

Because our sample microservices use Hydra, we'll need an accessible instance of Redis.  Let's look at two ways to address this requirement.

The first, and more production friendly method is to use a hosted Redis cluster, such as Amazon's ElasticCache for Redis or the [RedisLabs](https://redislabs.com/) service. The easiest approach will be to head over to RedisLabs and setup a free trial instance. The process takes a few minutes, and you'll end up with a Redis connection string that you can use with your test cluster. The connection string will look something like this: `redis-16122.c1.us-east-1-3.ec2.cloud.redislabs.com:16883` and you add that to your service's `config/config.json` file.

```
"redis": {
  "url": "redis://redis-16122.c1.us-east-1-3.ec2.cloud.redislabs.com:16883/0"
}
```

The second method is the one we saw in the first article in this series.  I'll recap the steps here.

First, sign into AWS and navigate over to the `EC2 Dashboard`. Once there click on the "Launch Instance" button. On the page that loads select the AWS Marketplace tab. You should see a screen like this:

> <img src="./AWS-choose-ami.png"/>

Search for `ECS Optimized` to locate the Amazon ECS-Optimized AMI. Amazon created this image for use with its EC2 Container Service.

For now, select the ECS-Optimized AMI and create an EC2 t2.micro instance.

There are a few things you'll want to do:

1. Use the Network VPC you selected earlier when you set up the `create-node` shell script
2. Set Auto-assign Public IP to `Enabled`
3. Before launching, you'll want to create a security group that allows you to SSH (naturally) and opens the default Redis port (6379) restricted to your laptop.  The port will be useful for testing.

You can choose the defaults for the remaining options.

Once the EC2 instance is ready, you can SSH into it to install a Redis container.  The following command adds Docker to the ec2-user group and creates a root folder called data, where our Redis data will persist. Finally, we use Docker to pull the Redis 3.0.7 container.

```shell
$ sudo usermod -a -G docker ec2-user
$ sudo mkdir /data
$ docker pull redis:3.0.7                    
```

Next we need to edit the /etc/rc.local file:

```shell
$ sudo vi /etc/rc.local
```

and append the following lines:

```
docker rm -f redis
docker run -d -p 6379:6379 --restart always -v /data:/data --name redis redis:3.0.7
```

After saving your changes, you can bounce the box: `sudo reboot`. On restart, your machine should be running a Redis instance.

Now, I know what you're thinking! - *"I should have used RedisLabs"* . But seriously, it's not too bad. Besides, using the above method, you'll be able to add other resources such as databases. The resources won't live in our Docker cluster but will be accessible within the same VPC. Again, this is an excellent way to test our cluster, but not recommended for production use.

### Testing the Redis setup

You can test access to your Redis instance by obtaining the remote IP address from the EC2 Dashboard.

If you have `redis-cli` installed you can connect to the instance using:

```shell
$ redis-cli -h 52.3.201.66
```

If you don't have redis-cli installed you can use telnet to interact with Redis:

```shell
$ telnet 52.3.201.66 6379
```

Then type: `info`. If you received an output listing instead of a connection closed message, then Redis is running.

## Creating and configuring the swarm

We're now ready to set-up our swarm. This process will involve creating a swarm manager and assigning workers. We begin configuring our swarm by requesting the external IP address of our the master01 node.

```shell
$ docker-machine ip master01
35.128.252.201
```

We'll use the machine's IP to initialize our swarm.

```shell
$ docker-machine ssh master01
$ sudo docker swarm init --advertise-addr 35.128.252.201:2377
Swarm initialized: current node (f15m9npvwumliqoe6wzor8tvh) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-2ohfpmuvx34e2o7wzag1qcohoti8layd0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
    35.128.252.201:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
    35.128.252.201:2377
```

We have two other master nodes to turn into managers.  Sadly, they won't get a pay raise.

```shell
$ docker-machine ssh master02
$ sudo docker swarm join \
--token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
35.128.252.201:2377
$ exit

$ docker-machine ssh master03
$ sudo docker swarm join \
--token SWMTKN-1-3ohfpmuvx39e2o7wzqg1qdohoti8layg0vk7ivoebncmw37p9y-07zcw2jht968k1td1f8dofcha \
35.128.252.201:2377
$ exit
```

From any swarm manager node you can view the status of managers:

```shell
$ sudo docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
f15m9npvwumliqoe6wzor8tvh *  master01  Ready   Active        Leader
t77rsrfdrq9u3v4rftldyzsgj    master02  Ready   Active        Reachable
ye7iq8hswgacvkz8il51v6je1    master03  Ready   Active        Reachable
```
Here we see that our master01 node is the leader, but should something happen to it - one of the other managers will be elected the new leader.  If our master01 node later recovers from its untimely accident, it won't resume as the leader, however it will be marked as reachable and eligible for promotion should something happen to another master node.

Now we're ready to configure our worker nodes.

```shell
for i in 1 2 3 4 5
do
  docker-machine ssh worker0$i sudo docker swarm join \
  --token SWMTKN-1-2ohfpmuvx34e2o7wzag1qcohoti8layd0vk7ivoebncmw37p9y-ezvmn0oj8a2o1l25l4fyahcn6 \
  35.128.252.201:2377
done
```

From a manager node, we can see the status of our swarm cluster. We see that our master01 node is the leader, with two managers reachable and waiting in the wings for their shot at a promotion.  We also see that none of our worker nodes are managers.

```
$ sudo docker node ls -t "30"
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
8caeo3nvjfa5d3jrqamciyijv    worker04  Ready   Active
c4nc3wnr45ii53hli5yomw234    worker03  Ready   Active
dfjrl5767thytai4lz9dfk360    worker05  Ready   Active
f15m9npvwumliqoe6wzor8tvh *  master01  Ready   Active        Leader
fcvzbgziv3ptso1r9egazizqv    worker01  Ready   Active
t77rsrfdrq9u3v4rftldyzsgj    master02  Ready   Active        Reachable
vz489z1vywrthlt4r9bw94zda    worker02  Ready   Active
ye7iq8hswgacvkz8il51v6je1    master03  Ready   Active        Reachable
```

### Swarm networking

At this stage, we have EC2 instances participating in a swarm as either managers or workers. We're now ready to create a network on which each node can communicate. In the containerization world, we call this an overlay network.

```shell
$ docker network create servicenet \
  --driver overlay \
  --subnet 10.0.9.0/24
```

You can list available networks with:

```shell
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
7ffba041b5b9        bridge              bridge              local
90d25bc2e419        docker_gwbridge     bridge              local
7af9c7ddd8f6        host                host                local
p5f0bg197oia        ingress             overlay             swarm
e5f86522a1d0        none                null                local
z6vut7t9439u        servicenet          overlay             swarm
```

Notice that there are two overlay networks, `ingress` and our newly created `servicenet` - both have a scope of `swarm`.

Here is how we'll use these two overlay networks:

<img src="./networks.png" alt="networks" width="400px" />

The `ingress` network will be used to receive API and message requests to our service aware router.  The `servicenet` will only receive traffic from the service router and won't be accessible to the outside world.

| network | usage | scope |
| --- | --- | --- |
| docker_gwbridge | A network created by Docker to allow containers to connect to their host | local |
| ingress | Network created by Docker to externally expose services and utilize the routing mesh | swarm |
| servicenet | Overlay network we created to enable container to container communication | swarm |

### Swarm visualization service

Wouldn't it be great if we could visualize the services in our Docker swarm? Such a tool might allow us to see the distribution of our services across machines and perhaps we'd be able to see the status of individual services. Now, wouldn't it be great if such a tool came packaged as a container that we could drop into our swarm? Well, I have some good news! Mano Marks has created a handy [docker swarm visualizer](https://github.com/ManoMarks/docker-swarm-visualizer) that we'll install onto a master node. Again, the reason we selected a master node is that we want this container to be remotely accessible.

```shell
$ docker-machine ssh master01
$ docker service create \
  --name=viz \
  --publish=8080:8080/tcp \
  --update-delay 10s \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  manomarks/visualizer
```

To view it, make sure to open port 8080 on the master nodes using an AWS security group that restricts access to your IP address.

<img src="./Visualizer.png" alt="AWS security group" width="900px" />

## Configuration management revisited

Hydra-based applications are initialized using a JavaScript object which contains the service name, description, IP and Port information and the location of the Redis server that Hydra depends on. Most often that information is loaded from a remote config.json file. In the case of a containerized hydra-based application, you have the option of overriding the packaged config.json file with one mapped to a volume using the `-v` fragment in the example below:

```shell
docker run -d \
  --workdir=/usr/src/app \
  -p 1337:1337 \
  --restart always \
  --add-host host:$HOST \
  --add-host redis:$DBS \
  --name auth-svcs \
  -v /usr/local/etc/auth-svcs:/usr/src/app/config \
  someco/auth-svcs:0.2.7
```

This can work fine in dockerized deployments which use ECS optimized EC2 images.  You simply have to ensure that the config files are present on the machine before running the container.

However, this isn't convenient for use with Docker Swarm since you don't necessarily know what machine your container will run on.  And later adding new machines would mean copying over config files. That just won't do!

Starting with [hydra](https://github.com/flywheelsports/hydra) 0.15.10 and [hydra-express](https://github.com/flywheelsports/hydra-express)  0.15.11 your hydra service can request its config from your Redis instance. Naturally, that implies that you've loaded the config into Redis in the first place.

To do this, you'll need [hydra-cli](https://github.com/flywheelsports/hydra-cli) version 0.5.4 or greater.

```shell
$ hydra-cli cfg push hydra-router:1.0.12 config.json
```

You're expected to provide the service name separated by a version string and a local config.json file whose contents will be uploaded.

Later you can retrieve a stored config using:

```shell
$ hydra-cli cfg pull hydra-router:1.0.12 > config.json
```

This is useful when you want to make changes to an existing config file or when you'd like to upload a new config based on an older copy.

It's worth pointing out that you can still build your microservice with a baked-in config file which has hardcoded entries to the resources your service needs. It's really up to you and the level of automation and flexibility you're after.

## Services

We can now use the Docker `service create` command to push containers into our swarm. In the example below we specify `--env HYDRA_REDIS` to point to the Redis server the service will use to retrieve its configuration file.  In production, the Redis instance would likely be an Amazon Elastic Cache cluster or one at RedisLabs.

```shell
$ docker service create \
    --name hydra-router \
    --network servicenet \
    --restart-condition any \
    --restart-max-attempts 5 \    
    --update-delay 10s \
    --constraint=node.role==manager \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hydra-router:1.0.12" \
    --publish 80:80 \
    --replicas=3 \
    flywheelsports/hydra-router:1.0.12
```

> A service is added to the ingress network when you use `-p` or `--publish`. The act of publishing a port indicates you want the container to be remotely accessible.

```shell
$ docker login
$ docker service create \
    --name hello-service \
    --network servicenet \
    --restart-condition any \
    --restart-max-attempts 5 \    
    --update-delay 10s \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hello-service:0.0.2" \
    --replicas=5 \
    cjus/hello-service:0.0.7
```

> Creating a service which does not use `-p` or `--publish` places the service in the `servicenet`, our private subnet. The service can still listen on a port for inter-service communication.

Both the hydra-router and hello-service containers above are publicly available - if you'd like to try this yourself.

#### Working with private containers

It's likely that at some point you'll need to use private containers for one or more of your services. To do this, you first sign into a master node and then issue a `docker login` command.

```shell
$ docker login
```

You can then issue the `docker service` command with the `--with-registry-auth` flag to tell Docker to use the credential you provided during the login.

Here's the full command:

```shell
$ docker service create \
    --name hello-service \
    --network servicenet \
    --update-delay 10s \
    --restart-condition any \
    --restart-max-attempts 5 \        
    --with-registry-auth \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="my-private-service:0.0.8" \
    --replicas=5 \
    cjus/my-private-service:0.0.8
```

### Removing services

You can remove services using:

```shell
$ docker service rm hydra-router
$ docker service rm hello-service
```

### Scaling services

One of the great benefits of using Docker Swarm mode is that you're able to perform other orchestration tasks such as scaling the number of services based on a container type.

Scaling services is a matter of using the Docker `service scale` command and specifying the service name and the number of required replicas.  This allows you to scale a service up or down.

```shell
$ docker service scale hydra-router=3
```

```shell
$ docker service scale hydra-router=0
```

### Updating services

You might be wondering what happens when you need to update a running service. Swarm mode allows you to update a running service using the `service update` command:

```
$ docker service update \
    --image flywheelsports/hydra-router:1.0.12 \
    hydra-router
```

To view the versions of your running containers you can use the Docker `service ls` command:

```shell
$ docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
1fs4uji2vs3j  offers-service  replicated  1/1       flywheelsports/offers-service:0.2.1
4r5tbyrmtvi2  hello-service   replicated  1/1       cjus/hello-service:0.0.5
qw7w325zg9e1  hydra-router    replicated  1/1       flywheelsports/hydra-router:1.0.9
tan1qxhlu8sj  viz             replicated  1/1       manomarks/visualizer:latest
```

## A quick test drive

To try all of this out, you'll need to obtain the DNS address of your Amazon ALB from the AWS dashboard.

<img src="./EC2_AEB_Console.png" alt="networks" width="700px" />

You can direct traffic to the load balancer doing something like this:

<img src="./hydra-alb-test_us-east-1_elb_amazonaws_com_v1_hello_test.png" alt="networks" width="700px" />

Refreshing the browser page would display different service IDs as the traffic is load balanced to our five hello services. It's worth pointing out the Amazon ALB is load balancing to one of our three HydraRouters which in-turn are load balancing to available hello services. 

## The Pièce De Résistance

As one of our [part-one](https://community.risingstack.com/deploying-node-js-microservices-to-aws-using-docker/) readers pointed out, and I'm paraphrasing here: *"It's not a microservices party until services are speaking with one another"* While that's a matter of opinion - it tends to be somewhat true in real world parties. The callout is an important one and the subject of our next and last example.

In an earlier RisingStack post we looked at a [silly little microservices game](https://community.risingstack.com/building-a-microservices-example-game-with-distributed-messaging/) called Hot Potato. In that post, we looked at inter-service messaging using Hydra. Each microservice instance acted as a single player and communicated with other instances to pass a virtual hot potato (aka JSON object) to other services. In the end, the player left holding the hot potato is declared the loser. Yes, it's slightly different from the classic children's games - tailored for services if you will.

We'll grab the code from the [earlier repo](https://github.com/cjus/hydra-hpp) and update it for use with Docker Swarm. You can view the resulting code [here](https://github.com/cjus/hpp-service).

Our new hot potato service has a single endpoint `/v1/hpp/startgame` which will cause the service which receives that request to start a new game. Internally, the hpp-service instances will use hydra messaging (built on redis Pub/Sub) to send non-http messages to one another.

### Configuring our player service

```javascript
{
  "environment": "development",
  "hydra": {
    "serviceName": "hpp-service",
    "serviceIP": "",
    "servicePort": 9000,
    "serviceType": "game",
    "serviceDescription": "Plays hot potato game",
    "redis": {
      "url": "redis://10.0.0.154:6379/15"
    }
  }
}
```

After modifying the default config.json file to include the location of our Redis instance we're now ready to upload the config to Redis using the hydra-cli app.

```shell
$ hydra-cli cfg push hpp-service:0.0.1 config.json
```

Now we're ready to launch player instances.

### Launching player instances

We'll launch containers the same way we've done earlier. In this case, we'll specify five instances using the replicas option.

```shell
$ docker service create \
    --name hpp-service \
    --network servicenet \
    --restart-condition any \
    --restart-max-attempts 5 \
    --update-delay 10s \
    --constraint=node.role==worker \
    --env HYDRA_REDIS_URL="redis://10.0.0.154:6379/15" \
    --env HYDRA_SERVICE="hpp-service:0.0.1" \
    --replicas=5 \
    cjus/hpp-service:0.0.1
```

You should then see the new `hpp-service` instances appear in the swarm visualizer. 

![](./Visualizer2.png)

### Starting a game!

To start a game we need to access the ALB with the route of our Hot Potato Service. The game runs for about 15 seconds, so we have to wait a bit for a reply. The IDs listed in square brackets are the Hydra service instance IDs for the services which participated in the game. You might be wondering why we only see three here? The reason is that the game is time limited with built-in delays so you'd have to increase the game duration to see more nodes participating. Running the game a second time should reveal new nodes.

![](./swarm_hpp_startgame.png)

To prove that this is actually working we can ask the API Gateway (HydraRouter) for a list of service nodes. Using the returned JSON, we can locate each of the instances which participated in the game.

![](./hydra-alb-us-east-1_elb_v1_router_list_nodes.png)

## Wrap-up

In this article, we stepped through creating a Docker Swarm cluster on AWS. We created and deployed microservices built using Hydra - which adds a microservice layer above ExpressJS. We used the [Hydra-Router](https://github.com/flywheelsports/hydra-router) as a service-aware API Gateway to route calls to our microservices without knowing their location within the swarm. And, our Hot Potato game service demonstrated inter-service messaging within the cluster.

This concludes our two-part series. However, this isn't an end - but rather a small beginning. Node-based microservices and containerization continue to be an evolving story. 

Containers used in this article can be found [here](https://hub.docker.com/u/cjus/) and [here](https://hub.docker.com/u/flywheelsports/). You can also review the code for the [hello-service](https://github.com/cjus/hello-service) and [hot potato service](https://github.com/cjus/hpp-service). If you have questions ping me on [twitter](https://twitter.com/cjus).
