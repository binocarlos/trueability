# Flocker tutorial (simple)

**This tutorial takes roughly 15 minutes**

Flocker is an open-source Container Data Volume Manager for your Dockerized application. 
It gives ops teams the tools they need to run containerized stateful services like databases in production.

Unlike a Docker data volume which is tied to a single server, a Flocker data volume, called a dataset, is portable and can be used with any container, no matter where that container is running in your Flocker cluster.

When you use the Flocker API or CLI to manage your stateful microservice, your volumes will follow your containers when they move between different hosts in your cluster.  

Being able to move a container and its data volume together between hosts is useful if you ever need to migrate your database to a more powerful host, relocate an application after server failure or migrate between environments. 

This tutorial will introduce you to the basic concepts of Flocker.
You will learn:

* How to control your Flocker cluster using the Flocker CLI
* How Flocker uses simple Deployment and Application YML files to define and deploy stateful applications
* How to move containers and their data between hosts as a single unit

## Before You Begin
A 3-node Flocker cluster has been provisioned and configured for you in a hosted environment so that you can try Flocker without having to do any setup work.

You will be controlling your Flocker cluster via the CLI that has been pre-installed for you. We have also pre-installed some files on the node running the CLI for use throughout the tutorial.  In order to access these files, make sure your command line is in the proper directory.  To access this directory, run:

```bash
$ cd /flocker-tutorials/tutorial-1
```

## Details of your sample application

This tutorial will use a simple two-container application to demonstrate the basic features of Flocker.  One container has a Python web application and the other has a Redis server, which stores its data on a volume.  Each time the Python web application serves a request, a counter is incremented and the number of visits is stored in Redis.  You will use Flocker to migrate the Redis container with its data volume from one host to another.  If this migration is successful, you will see the correct visit count in the Redis database.

![initial setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-initial-setup.svg "In the initial server-side Flocker setup there are two servers, one of which has two Docker containers running; one container is a running a web application, the other has a Redis database with a volume.")
The following diagram illustrates how the server-side Flocker setup will be configured at the end of the tutorial:

![final setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-final-setup.svg "Following the completion of this tutorial the server-side Flocker setup will be configured with the web application still running within a container on the first server, while the Redis server with a volume is running on the second server.")
Let's get started!

##a Flocker data volume, Deploying an Application on the First Host
The first step is to create the two Docker containers on one of the hosts.
One container has a Python web application (Flask) and the other has a Redis server, which stores its data on a volume.

The `docker-compose.yml`, or the Application file, describes your distributed application using the Docker Compose format:

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

The `deployment-node1.yml` file, or the Deployment file, describes which containers to deploy, and where. In this case, we are going to have Flocker deploy both the Flask container and the Redis container to the same host:

```yaml
"version": 1
"nodes":
  "1.2.3.4": ["web", "redis"]
  "5.6.7.8": []
```

Both of these YML files have been pre-installed for you on your CLI node.

To deploy the application, run the following command:

```bash
root@clinode:~$ flocker-deploy control-service deployment-node1.yml docker-compose.yml
```
Let's look at what we just did.

The `flocker-deploy` command takes 3 arguments:  

* the IP address of the Flocker Control Service (in this case we've saved this as a configure variable that we reference with the argument `control-service`)
* the Deployment yaml file and
* the Application yaml file

By running 

```bash
$ flocker-deploy control-service deployment-node1.yml docker-compose.yml
```
we've taken the application described in our Application file and Ddployed it to the hosts in our Deployment file.

Now let's check if our application is available on the hosts that we've deployed it to.

* Visit http://1.2.3.4/.
  You will see the visit count displayed.
* Visit http://5.6.7.8/.
  You will also see that the count persists because Flocker routes the traffic from either node named in the Deployment file to the one that has the application.  This makes it possible to move your containers and their volumes around the cluster without having to update any DNS or application settings.

## Migrating a Container to the Second Host

The diagram below illustrates your current server-side Flocker setup after running `flocker-deploy`:
![initial setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-initial-setup.svg "In the server-side Flocker setup there are two servers, one of which has two Docker containers running; one container is a running a web application, the other has a Redis database with a volume.")
Now, let's move the Redis container and its data volume to the other cluster node.

To do this, we'll simply update the Deployment file to put the Redis container on node `5.6.7.8`

```yaml
"version": 1
"nodes":
  "1.2.3.4": ["web"]
  "5.6.7.8": ["redis"]
```

and rerun `flocker-deploy` using the `deployment-node2.yml` file:

```bash
root@clinode:~$ flocker-deploy control-service deployment-node2.yml docker-compose.yml
```

The container on the Redis server and its volume have now both been moved to the second host, and Flocker has maintained its link to the web application on the first host:

* Visit http://1.2.3.4/.
  You will see the visit count is still persisted on this node even though the application is no longer running on that host
* Visit http://5.6.7.8/.
  You will see that the count still persists, even though the container with the volume has moved between hosts.

## Result
Using Flocker, you just moved a Docker container with its volume, while persisting its link to a web app on another server.

The following diagram illustrates how your server-side Flocker setup looks now:
![final setup](https://rawgithub.com/binocarlos/trueability/master/tutorials/images/flocker-tutorial-final-setup.svg "The web application is still running within a container on the first server, while the Redis server with a volume is now running on the second server.")

## Next steps
The tutorial above demonstrated the most basic features of Flocker.  If you want to learn more about how Flocker works, follow along with an In-depth tutorial [link] in your live demo environment or try deploying and migrating your own Dockerized application.  If you want to install Flocker in your own environment, follow along with our Installation guide [link].

## Cleanup

Once you have run this tutorial - it can be useful to clean up the Flocker cluster so you can do other tutorials without the containers and data from this tutorial getting in the way.

We have included a `deployment-reset.yml` file that you can use to remove the containers from the cluster.

```bash
$ flocker-deploy control-service deployment-reset.yml docker-compose.yml
```

You can also take steps to reset the cluster fully (including removing all data volumes etc).

You can use [this guide](http://build.clusterhq.com/results/docs/master/build-7878/using/administering/cleanup.html) to do so.
