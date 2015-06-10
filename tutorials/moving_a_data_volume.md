# Tutorial (simple)

**This tutorial takes roughly 10 minutes**

Flocker is an open-source data volume manager for your Dockerized application. 
It gives ops teams the tools they need to run containerzied stateful services like databases in production.

Unlike a Docker data volume which is tied to a single server, a Flocker data volume, called dataset, is portable. and can be used with any container, no matter where that container is running in your Flocker cluster.

When you use the Flocker API or CLI to manage your stateful microservice, your volumes will follow your containers when they move between different hosts in your cluster.  

Being able to move a container and its data volume together between hosts is useful if you ever need to migrate your database to a more powerful host, relocate an application after server failure or migrate between environments. 

This Tutorial will introduce you to the basic concepts of Flocker.
You will learn:

* How to control your Flocker cluster using the Flocker CLI
* How Flocker uses simple Deployment and Application YML files to define and deploy stateful applications
* How to move containers and their data between hosts as a single unit

You will use Flocker to migrate a Docker container with its data volume from one host to another.
The container you move will be part of a two-container application, the other container will not move and the two will remain connected even when they are on different hosts.

## Before You Begin
A 3-node Flocker cluster has been provisioned & configured for you in a hosted environment so that you can try Flocker without having to do any setup work.

You will be controlling your Flocker cluster via the CLI that has been pre-installed for you. We have also pre-installed some files on the node running the CLI for use throughout the Tutorial.  In order to access these files, make sure your command line is in the proper directory:

```bash
$ cd /flocker-tutorials/tutorial-1
```

##

![initial setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-initial-setup.svg "In the initial server-side Flocker setup there are two servers, one of which has two Docker containers running; one container is a running a web application, the other has a Redis database with a volume.")

The following diagram illustrates how the server-side Flocker setup will be configured at the end of the tutorial:

![final setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-final-setup.svg "Following the completion of this tutorial the server-side Flocker setup will be configured with the web application still running within a container on the first server, while the Redis server with a volume is running on the second server.")


We have uploaded the templates used in this tutorial into the `/flocker-tutorials/tutorial-1` folder.  The first step is to change directory into this folder:




Flocker manages the data migration and the link between the two containers.

If you have any feedback or problems, you can [talk to us](https://clusterhq.com/about/)

Let's get started!

### Deploying an Application on the First Host
The first step is to create two Docker containers on one of the hosts.
One container has a Python web application and the other has a Redis server, which stores its data on a volume.

The `fig.yml` file describes your distributed application:

```yaml
web:
  image: clusterhq/flask
  links:
   - "redis:redis"
  ports:
   - "80:80"
redis:
  image: dockerfile/redis
  ports:
   - "6379:6379"
  volumes: ["/data"]
```

The `deployment-node1.yml` file describes which containers to deploy, and where:

```yaml
"version": 1
"nodes":
  "1.2.3.4": ["web", "redis"]
  "5.6.7.8": []

```

**note** the IP addresses in your version of `deployment-node1.yml` will be different to those shown above.

Now you can use the Flocker CLI to deploy both your web application and server onto one of the Flocker nodes.

```bash
root@clinode:~$ flocker-deploy control-service deployment-node1.yml fig.yml
```

* Visit http://1.2.3.4/.
  You will see the visit count displayed.
* Visit http://5.6.7.8/.
  You will see that the count persists because Flocker routes the traffic from either node named in the deployment file to the one that has the application.

**note** the IP addresses for your setup will be different than above - please change `1.2.3.4` and `5.6.7.8` to be what you have set it to in `deployment-node1.yml`

Migrating a Container to the Second Host
========================================

The diagram below illustrates your current server-side Flocker setup:

![initial setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-initial-setup.svg "In the server-side Flocker setup there are two servers, one of which has two Docker containers running; one container is a running a web application, the other has a Redis database with a volume.")

To move the container with the Redis server along with its data volume, use the `deployment-node2.yml` file:

:download:`deployment-node2.yml`

```yaml
"version": 1
"nodes":
  "1.2.3.4": ["web"]
  "5.6.7.8": ["redis"]
```

**note** the IP addresses in your version of `deployment-node2.yml` will be different to those shown above.

Now you can use the Flocker CLI to migrate one of the containers to the second host:

```bash
root@clinode:~$ flocker-deploy control-service deployment-node2.yml fig.yml
```

The container on the Redis server and its volume have now both been moved to the second host, and Flocker has maintained its link to the web application on the first host:

* Visit http://1.2.3.4/.
  You will see the visit count is still persisted.
* Visit http://5.6.7.8/.
  You will see that the count still persists, even though the container with the volume has moved between hosts.

**note** the IP addresses for your setup will be different than above - please change `1.2.3.4` and `5.6.7.8` to be what you have set it to in `deployment-node2.yml`

Result
======

Using Flocker, you just moved a Docker container with its volume, while persisting its link to a web app on another server.

The following diagram illustrates how your server-side Flocker setup looks now:

![final setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-final-setup.svg "The web application is still running within a container on the first server, while the Redis server with a volume is now running on the second server.")