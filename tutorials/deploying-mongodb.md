# Flocker tutorial (In-depth)

**This tutorial takes roughly 30 minutes**

In the simple Flocker tutorial [link], you learned about the basic features of Flocker including its CLI, Application and Deployment files and the `flocker-deploy` command.  
The goal of this tutorial is to teach you more about Flocker's container, network, and volume management functionality.

In this Tutorial you will learn how to:

* Expose ports so that any containers is reachable on any container host
* Specify volume mount points for containers

This tutorial is based around the setup of a MongoDB service.
Flocker is a generic Container Data Volume manager.
MongoDB is used only as an example here.
Any application you can deploy into Docker you can manage with Flocker.


## Before You Begin
A 3-node Flocker cluster has been provisioned and configured for you in a hosted environment so that you can try Flocker without having to do any setup work.

You will be controlling your Flocker cluster via the CLI that has been preinstalled for you. We have also pre-installed some files on the node running the CLI for use throughout the tutorial. In order to access these files, make sure your command line is in the proper directory. To access this directory, run:

```bash
$ cd /flocker-tutorials/tutorial-2
```

### Deploying an Application
Let's look at an extremely simple Flocker configuration for one node running a container containing a MongoDB server.

Application configuration: `minimal-application.yml`
```yaml
"version": 1
"applications":
  "mongodb-example":
    "image": "clusterhq/mongodb"
```

Deployment configuration: `minimal-deployment.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": ["mongodb-example"]
  "5.6.7.8": []
```

Notice that in the Deployment configuration file, we mention that the second node that has no applications deployed on it to ensure that `flocker-deploy` knows that it exists.
If we don't do this, certain actions that might need to be taken on that node in the future will not happen, e.g. stopping currently running applications.
It is thus important to list all of the nodes in your Flocker cluster in the Deployment file, even if you do not want to use them currently.

At this point, we haven't done anything yet.  Take a look at what containers Docker is running on node 1 by running:

```bash
$ ssh root@1.2.3.4 docker ps
```

Looking at the command line output, we can see that no containers are running on node 1. 

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS     PORTS     NAMES
$
```

To deploy our Mongo on container to node 1, use `flocker-deploy` with the simple configuration files given above.  Run:

```bash
$ flocker-deploy control-service minimal-deployment.yml minimal-application.yml
```

`flocker-deploy` has made the necessary changes to make your node match the state described in the configuration files you supplied.

Remember, the `flocker-deploy` command takes 3 arguments:

* the IP address of the Flocker Control Service (in this case we've saved this as a configure variable that we reference with the argument control-service)
* the Deployment yaml file and
* the Application yaml file

We can check node `1.2.3.4` again.  Run:

```bash
$ ssh root@1.2.3.4 docker ps
```

The output shows MongoDB is running as expected:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-example
$
```
### Moving an Application
Let's see how `flocker-deploy` can move this application to a different VM.

This time we will use a different Deployment file that instructs Flocker to move the MongoDB container to another host.  In this case, we want MongoDB to be run on host [5.6.7.8], instead of [1.2.3.4]

Here is what our new Deployment file looks like:

`minimal-deployment-moved.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": []
  "5.6.7.8": ["mongodb-example"]
```
Nothing in the Application configuration file needs to change to move a container.  Moving the container and its data only requires an update to the Deployment configuration.

Run `flocker-deploy` again to enact the change:

```bash
$ flocker-deploy control-service minimal-deployment-moved.yml minimal-application.yml
$
```

If we do `docker ps` on node `1.2.3.4` we will see that no containers are running in this host anymore.  Run:

```bash
$ ssh root@1.2.3.4 docker ps
```

And inspect the output:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS     PORTS     NAMES
$
```

If we inspect `5.6.7.8`, we will see that MongoDB has been successfully moved. Run:

```bash
$ ssh root@5.6.7.8 docker ps
```

And inspect the output which shows MongoDB running in this host:
```bash
$ ssh root@5.6.7.8 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-example
$
```

At this point you have successfully deployed a MongoDB server in a container on your VM.
You've also seen how Flocker can move an existing container between hosts.
There's no way to interact with it apart from looking at the `docker ps` output yet.
In the next section of the tutorial you'll see how to expose container services on the host's network interface.

### Exposing Ports
Each application running in a Docker container has its own isolated networking stack. 
To communicate with an application running inside the container we need to forward traffic from a network port in the node where the container is located to the appropriate port within the container. 
Flocker takes this one step further: an application is reachable on all nodes in the cluster, no matter where it is currently located.

Let's start a MongoDB container that exposes the database to the external world.

`port-application.yml`
```yaml
"version": 1
"applications":
  "mongodb-port-example":
    "image": "clusterhq/mongodb"
    "ports":
    - "internal": 27017
      "external": 27017
```

`port-deployment.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": ["mongodb-port-example"]
  "5.6.7.8": []
```

We will once again run these configuration files with `flocker-deploy`.  Run:

```bash
$ flocker-deploy control-service port-deployment.yml port-application.yml
```

If we inspect node `1.2.3.4` we will see MongoDB running.  Run:

```bash
$ ssh root@1.2.3.4 docker ps
```

Now we see MongoDB running on host 1:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-port-example
$
```

This time we can communicate with the MongoDB application by connecting to the node where it is running.
Using the `mongo` command line tool which has been preinstalled in this live demo environment, we will insert an item into a database and check that it can be found.
If you get a connection refused error try again after a few seconds; the application might take some time to fully start up.

> To keep your download for the tutorial as speedy as possible, we've bundled the latest development release of MongoDB in to a micro-sized Docker image. *You should not use this image for production.*

```bash
$ mongo 1.2.3.4
MongoDB shell version: 2.4.9
connecting to: 172.16.255.250/test
> use example;
switched to db example
> db.records.insert({"flocker": "tested"})
> db.records.find({})
{ "_id" : ObjectId("53c958e8e571d2046d9b9df9"), "flocker" : "tested" }
```

We can also connect to the other node where it isn't running and the traffic will get routed to the correct node:

```bash
alice@mercury:~/flocker-tutorial$ mongo 5.6.7.8
MongoDB shell version: 2.4.9
connecting to: 5.6.7.8/test
> use example;
switched to db example
> db.records.find({})
{ "_id" : ObjectId("53c958e8e571d2046d9b9df9"), "flocker" : "tested" }
```

Since the application is transparently accessible from both nodes you can configure a DNS record that points at both IPs and access the application regardless of its location.

At this point you have successfully deployed a MongoDB server and communicated with it.
You've also seen how external users don't need to worry about applications' location within the cluster.
In the next section of the tutorial you'll learn how to ensure that the application's data moves along with it, the final step to running stateful applications on a cluster.

### Data Volumes

#### The Problem

By default moving an application from one node to another does not move its data along with it.
Before proceeding let's see in more detail what the problem is by continuing the :doc:`Exposing Ports <exposing-ports>` example.

Recall that we inserted some data into the database.
Next we'll use a new configuration file that moves the application to a different node.

`port-deployment-moved.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": []
  "5.6.7.8": ["mongodb-port-example"]
```

Run:

```bash
$ flocker-deploy control-service port-deployment-moved.yml port-application.yml
```

Then run:

```bash
$ ssh root@5.6.7.8 docker ps
```

We can see that Mongo is on host 2:

```bash
$ ssh root@5.6.7.8 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest     mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-port-example
$
``

If we query the database the records we've previously inserted have disappeared!
The application has moved but the data has been left behind.

```bash
$ mongo 5.6.7.8
MongoDB shell version: 2.4.9
connecting to: 5.6.7.8/test
> use example;
switched to db example
> db.records.find({})
>
```

#### The Solution

Unlike many other Docker frameworks Flocker has a solution for this problem, a Volume Manager.
An application with a Flocker volume configured will move the data along with the application, transparently and with no additional intervention on your part.

We'll create a new configuration for the cluster, this time adding a volume to the MongoDB container.

`volume-application.yml`
```yaml
"version": 1
"applications":
  "mongodb-volume-example":
    "image": "clusterhq/mongodb"
    "ports":
    - "internal": 27017
      "external": 27017
    "volume":
      # The location within the container where the data volume will be
      # mounted:
      "mountpoint": "/data/db"
```

`volume-deployment.yml`
```bash
"version": 1
"nodes":
  "1.2.3.4": ["mongodb-volume-example"]
  "5.6.7.8": []
```

Then we'll run these configuration files with `flocker-deploy`. Run:

```bash
$ flocker-deploy control-service volume-deployment.yml volume-application.yml
```

When we inspect node 1, we can see that MongoDB is there as expected.  Run:

```bash
$ ssh root@1.2.3.4 docker ps
```

And inspect the output:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-volume-example
$
```

Once again we'll insert some data into the database:

```bash
$ mongo 1.2.3.4
MongoDB shell version: 2.4.9
connecting to: 1.2.3.4/test
> use example;
switched to db example
> db.records.insert({"the data": "it moves"})
> db.records.find({})
{ "_id" : ObjectId("53d80b08a3ad4df94a2a72d6"), "the data" : "it moves" }
```

Next we'll move the application to the other node. 

`volume-deployment-moved.yml`
"version": 1
"nodes":
  "1.2.3.4": []
  "5.6.7.8": ["mongodb-volume-example"]
```

Run: 

```bash
$ flocker-deploy control-service volume-deployment-moved.yml volume-application.yml
```

Check to see if Mongo is running on the correct host.  Run:

```bash
$ ssh root@5.6.7.8 docker ps
```

And inspect the output:

```bash
$ ssh root@5.6.7.8 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-volume-example
$
```

This time however, because we've specified a volume mountpoint in the Application configuration file, the data has moved with the application:

```bash
alice@mercury:~/flocker-tutorial$ mongo 5.6.7.8
MongoDB shell version: 2.4.9
connecting to: 5.6.7.8/test
> use example;
switched to db example
> db.records.find({})
{ "_id" : ObjectId("53d80b08a3ad4df94a2a72d6"), "the data" : "it moves" }
```

At this point you have successfully deployed a MongoDB server and communicated with it.
You've also seen how Flocker allows you to move an application's data to different locations in a cluster as the application is moved.
You now know how to run stateful applications in a Docker cluster using Flocker.

### Next steps
The tutorial above demonstrated some more features of Flocker.  You have root access to this live demo environment.  Why don't you try deploying your own stateful application. If you want to install Flocker in your own environment, follow along with our Installation guide [link].

Once you have run this tutorial - it can be useful to clean up the Flocker cluster so you can do other tutorials without the containers and data from this tutorial getting in the way.

We have included a `deployment-reset.yml` file that you can use to remove the containers from the cluster.

```bash
$ flocker-deploy control-service deployment-reset.yml minimal-application.yml
```

You can also take steps to reset the cluster fully (including removing all data volumes and containers).

You can use [this guide](http://build.clusterhq.com/results/docs/master/build-7878/using/administering/cleanup.html) to do so.

If you have any feedback or problems, you can [talk to us](https://clusterhq.com/contact/)
