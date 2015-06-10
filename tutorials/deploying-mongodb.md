# Tutorial: Deploying and Migrating MongoDB

> **note** This tutorial takes roughly 30 minutes

The goal of this tutorial is to teach you to use Flocker's container, network, and volume orchestration functionality.
By the time you reach the end of the tutorial you will know how to use Flocker to create an application.
You will also know how to expose that application to the network and how to move it from one host to another.
Finally you will know how to configure a persistent data volume for that application.

This tutorial is based around the setup of a MongoDB service.
Flocker is a generic container manager.
MongoDB is used only as an example here.
Any application you can deploy into Docker you can manage with Flocker.

If you have any feedback or problems, you can [talk to us](https://clusterhq.com/about/)

## Before You Begin
A 3 node Flocker cluster has been provisioned for you.  Before you begin the tutorial, ensure that you have a command line on the Flocker CLI.

We have uploaded the templates used in this tutorial into the `/flocker-tutorials/tutorial-2` folder.  The first step is to change directory into this folder:

```bash
$ cd /flocker-tutorials/tutorial-2
```

### Starting an Application
Let's look at an extremely simple Flocker configuration for one node running a container containing a MongoDB server.

`minimal-application.yml`
```yaml
"version": 1
"applications":
  "mongodb-example":
    "image": "clusterhq/mongodb"
```

`minimal-deployment.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": ["mongodb-example"]
  "5.6.7.8": []
```

**note** the IP addresses in your version of `minimal-deployment.yml` will be different to those shown above.

Notice that we mention the node that has no applications deployed on it to ensure that `flocker-deploy` knows that it exists.
If we hadn't done that certain actions that might need to be taken on that node will not happen, e.g. stopping currently running applications.

Next take a look at what containers Docker is running on the VM you just created.
The node IPs are those which were specified earlier in the `Vagrantfile`:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE    COMMAND    CREATED    STATUS     PORTS     NAMES
$
```

From this you can see that there are no running containers.
To fix this, use `flocker-deploy` with the simple configuration files given above and then check again:

```bash
$ flocker-deploy control-service minimal-deployment.yml minimal-application.yml
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-example
$
```

`flocker-deploy` has made the necessary changes to make your node match the state described in the configuration files you supplied.

### Moving an Application
Let's see how `flocker-deploy` can move this application to a different VM.

This time we will use a different deployment file that instructs Flocker to move the MongoDB container to another host.

`minimal-deployment-moved.yml`
```yaml
"version": 1
"nodes":
  "1.2.3.4": []
  "5.6.7.8": ["mongodb-example"]
```

**note** the IP addresses in your version of `minimal-deployment.yml` will be different to those shown above.

Note that nothing in the application configuration file needs to change.
*Moving* the application only involves updating the deployment configuration.

Use `flocker-deploy` again to enact the change:

```bash
$ flocker-deploy control-service minimal-deployment-moved.yml minimal-application.yml
$
```

`docker ps` shows that no containers are running on `1.2.3.4`:

```bash
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
$
```

and that MongoDB has been successfully moved to `5.6.7.8`:

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
Each application running in a Docker container has its own isolated networking stack. To communicate with an application running inside the container we need to forward traffic from a network port in the node where the container is located to the appropriate port within the container. Flocker takes this one step further: an application is reachable on all nodes in the cluster, no matter where it is currently located.

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

We will once again run these configuration files with `flocker-deploy`:

```bash
$ flocker-deploy control-service port-deployment.yml port-application.yml
$ ssh root@1.2.3.4 docker ps
CONTAINER ID    IMAGE                       COMMAND    CREATED         STATUS         PORTS                  NAMES
4d117c7e653e    clusterhq/mongodb:latest    mongod     2 seconds ago   Up 1 seconds   27017/tcp, 28017/tcp   mongodb-port-example
$
```

This time we can communicate with the MongoDB application by connecting to the node where it is running.
Using the `mongo` command line tool we will insert an item into a database and check that it can be found.
You should try to follow along and do these database inserts as well.

> To keep your download for the tutorial as speedy as possible, we've bundled the latest development release of MongoDB in to a micro-sized Docker image. *You should not use this image for production.*

If you get a connection refused error try again after a few seconds; the application might take some time to fully start up.

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

