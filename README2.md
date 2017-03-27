# Deploying Node Microservices to AWS using Docker (part two)

In part one of this series we look at creating a simple microservice and packaging a microservice in a Docker container. We also looked at deploying the container to AWS using Amazon's ECS optimized Linux AMI - which has the Docker engine pre-installed.

In this post we'll create a Docker Swarm cluster almost entirely from the command line! In the process we'll deploy multiple services and introduce application and message-based load balancing.

The resulting architecture will be quite scalable - unless of course you're Netflix and have Netflix size problems.

In any case the approach we'll look at here can be further scaled in complexity to accommodate your specific needs.

Let's get started.

## Configuration management revisited

In the first post of this series, we considered how config files can be baked into a container. We also saw how we can use docker volume mapping to allow our container to reference external config files.

That all works fine - up to a point. A key goal with containerization is to create truly portable containers. Containers that ideally do not depend on the configuration of the machine it's running in. Furthermore, in a machine cluster we might not know or care on which machine in a cluster our container runs in. That ideal is comprised if we need to update each cluster machine with a copy of our service's configuration files.

Thus, the management of our service config files can quickly become a headache.

There are lots of ways to address configuration management, however we're going to use a feature built into Hydra.
