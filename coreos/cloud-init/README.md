# Deploying a Java 8 application using Spring Boot and MongoDB on a CoreOS cluster on Azure, using Docker and Fleet

This tutorial is part of [P@'s Linux on Azure series](/../../).

<img src="../../../master/coreos/cloud-init/spring-doge/spring-doge-overview.jpg"/>

## Getting a CoreOS cluster up and running on Azure

CoreOS for Azure was [released on 10.20.2014](http://azure.microsoft.com/blog/2014/10/20/azures-getting-bigger-faster-and-more-open/). The CoreOS website documentation [Running CoreOS on Azure](https://coreos.com/docs/running-coreos/cloud-providers/azure/) is a good start.
@timfpark also wrote [a great tutorial for how to setup a CoreOS cluster on Azure](https://github.com/timfpark/coreos-azure), and deploy a simple NodeJS app. This tutorial is quite similar for the CoreOS aspect, adding a few details about remote fleet setup, and shows deployment of a Java Spring application using MongoDB. CoreOS documentation includes a nice overview of [CoreOS Cluster Architectures](https://coreos.com/docs/cluster-management/setup/cluster-architectures/), describing various cluster sizes. This tutorial explains how to provision a small CoreOS cluster in Azure. I will add documentation to create the other topologies later, but these instructions should give you enough knowledge to create them by yourself.

<img src="https://coreos.com/docs/cluster-management/setup/cluster-architectures/small.png"/>

<img src="https://coreos.com/docs/cluster-management/setup/cluster-architectures/dev.png"/>

<img src="https://coreos.com/docs/cluster-management/setup/cluster-architectures/prod.png"/>

### Automated provisioning script

The instructions below go into details on how to provision a CoreOS cluster manually using the Azure Cross Platform CLI, and tell you about all the configuration parameters. I automated all that in a Python script that lets you provision a cluster in one shot: [Azure CoreOS cluster deployment tool](https://github.com/chanezon/azure-linux/blob/master/coreos/cluster/README.md). You may want to use that instead of the manual approach.

### Create your cloud-init config file

Modify cloud-init.yml with https://discovery.etcd.io/new discovery url, ssh key, name and hostname for each of the hosts you want to create. Here is an example [https://github.com/chanezon/azure-linux/blob/master/coreos/cloud-init/cloud-init.yml](cloud-init.yml) file. Then create the VMs. These commands create ssh endpoints for easier debugging.

A common mistake is to reuse discovery urls from previous cluster: think about creating a new one when creating a new cluster.

### A note on ssh keys

If you follow the [Azure documentation to create your ssh keys](http://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-use-ssh-key/), using openssl, you will end up with a ~/.ssh/my_private_key.key and ~/.ssh/my_public_key.pem files. openssh understands the private key format from openssl, but the [public key format for openssl and openssh are different](http://security.stackexchange.com/questions/32768/converting-keys-between-openssl-and-openssh). When you provision a VM, Azure will do the conversion for you and place the ssh public key in yourusername/.ssh/authorized_keys. You need that public ssh key in the ssh format in your cloud-init.yml.
In order to generate the public key in openssh format from your openssl cert, you need to use ssh-keygen on your private key.

```shell
ssh-keygen -y -f ~/.ssh/my_private_key.key
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDOC4ZPy3a+F/DRQefLG/IteM00PYpJlc4Mga7S2mLv86aglqfAgXTHxXHho3ggfRRlxvLnj
```
Use that as the value for the ssh_authorized_keys: entry in your cloud-init file.
What this will do is that it will add this key in authorized_keys for user core, in /home/core/.ssh/authorized_keys.

### Create several CoreOS VMs for your cluster

Create one cloud-init yml file for each VM you want to create. For now the only CoreOS image available is in the alpha channel: 2b171e93f07c4903bcad35bda10acf22__CoreOS-Alpha-475.1.0. What's really cool about this is that this comes with Docker 1.3, fresh from last week. Check [CoreOS on Azure documentation](https://coreos.com/docs/running-coreos/cloud-providers/azure/) for different options as they appear. Create he VMs using the following command:
```shell
azure vm create -l "West US" --ssh --ssh-cert ~/.ssh/<Your pem file for ssh> <hostname> <imagename>  --virtual-network-name <virtual network name> username password --custom-data <cloud-init-file.yml>
```

Then check your cluster:
```shell
ssh username@hostname.cloudapp.net
sudo etcdctl ls --recursive
sudo fleetctl list-machines
MACHINE		IP		METADATA
415162a4...	10.0.0.5	region=us-west
7c7f60e0...	10.0.0.4	region=us-west
```

Then you can start playing with fleet on your cluster.

In this example cloud-init file, I use $private_ipv4, which will work for an Azure only cluster: all VMs being in the same virtual network, they can talk to each other. If you want to create a multi-cloud cluster, or connect to fleet from a remote machine, follow instructions at https://coreos.com/docs/cluster-management/setup/cloudinit-cloud-config/, use $public_ipv4 and open an endpoint on port 4001 from Azure.

```shell
azure vm endpoint create <hostname> 4001 4001
```

Have fun with CoreOS on Azure!

### Configuring a remote fleet client

If you want to manage your cluster with fleet from a remote client, on a Mac, you need to build fleetctl on Mac.

```shell
git clone https://github.com/coreos/fleet.git
cd fleet/
./build
cp bin/fleetctl /usr/local/bin
```

fleetctl client tool uses SSH to interact with a fleet cluster (cf https://github.com/coreos/fleet/blob/master/Documentation/using-the-client.md#remote-fleet-access & https://github.com/coreos/fleet/blob/master/Documentation/deployment-and-configuration.md). You need to configure ssh to use the private key for the public key you specified in cloud-init file. By default, fleetctl uses the core user for ssh tunneling. [fleetctl issue 536](https://github.com/coreos/fleet/issues/536) adds a --ssh-username flag but it has not been merged yet. So you In order for fleetctl to use your private key, you need to use [ssh-add](http://linux.die.net/man/1/ssh-add) and ssh-agent for your session.
```shell
ssh-add ~/.ssh/my_private_key.key
fleetctl --tunnel <hostname>.cloudapp.net:22 list-machines
MACHINE IP METADATA
415162a4... 10.0.0.5 region=us-west
7c7f60e0... 10.0.0.4 region=us-west
```

Now you can start using fleet from your dev machine to manage services on your cluster.

### A more production-like cluster

#### cloud service, vnet

In the previous example, each VM is in its own cloud service, and has ssh port 22 open. This makes it convenient for testing, but one you deploy an application, you want to use Azure for load balancing. A more typical deployment would have one cloud service per tier, or cluster, using an Azure load balanced set to load balance between instances. The initial version of this documentation recommended to deploy your VMs in an affinity group. Since then, I discovered [@mknz](https://twitter.com/mknz)'s excellent [post on why affinity groups in Azure are not useful anymore](http://convective.wordpress.com/2014/07/11/affinity-groups-in-azure/). Don't use an affinity groups. The Azure documentation on [About Regional VNets and Affinity Groups for Virtual Network](http://msdn.microsoft.com/en-us/library/azure/jj156085.aspx) is also relevant. However, the Azure Cross Platform CLI has [a still open bug that does not let you create a regional VNet without an affinity group](https://github.com/Azure/azure-sdk-tools-xplat/issues/1255). The current behavior of the CLI is to fetch your existing affinity groups for the region, and use the first one, or create a new one. So you have 3 choices: create a regional VNet in the portal (good), use the CLI and specify an affinity group (suboptimal), or don't use a subnet for your cloud service (ok if you run only one type of service in the cloud service). In this tutorial I recommend using the portal.

We start by creating a cloud service.
```shell
azure service create --location "West US" <cloud-service-name>
```
When [issue 1255](https://github.com/Azure/azure-sdk-tools-xplat/issues/1255) is fixed, you will be able to create a vnet and subnet using this command. Here I create a subnet for frontend roles. Later, this leaves me an option to add a different subnet for backend roles.
```shell
azure network vnet create <vnet-name> \
--location "West US" \
--address-space 192.168.0.0 \
--cidr 16 \
--subnet-name "frontend" \
--subnet-start-ip 192.168.0.4 \
--subnet-vm-count 256
```

For now, [please do that in the portal](http://azure.microsoft.com/en-us/documentation/articles/create-virtual-network/). Screenshots below.

<img src="../../../master/coreos/cloud-init/vnet/vnet-1.png"/>

<img src="../../../master/coreos/cloud-init/vnet/vnet-2.png"/>

<img src="../../../master/coreos/cloud-init/vnet/vnet-3.png"/>

There is a workaround to create your vnet using the x-platform cli: use the option to export all your network config in a json file, add your new subnet config in the file, and import it.
```shell
azure network export net-config.json
```
Generates a file like this:
```json
{
	"VirtualNetworkConfiguration": {
		"Dns": "",
		"VirtualNetworkSites": [{
			"AddressSpace": ["192.168.0.0/20"],
			"Subnets": [{
				"AddressPrefix": "192.168.0.0/23",
				"Name": "Subnet-1"
			}],
			"Name": "pat-coreos-13-network",
			"Location": "West US"
		},
		... more vnets
			}]
	}
}
```
Make a copy called net-config-new.json, and add your vnet config in there. The section you add looks like this:
```json
    {
			"AddressSpace": ["192.168.0.0/16"],
			"Subnets": [{
				"AddressPrefix": "192.168.0.4/24",
				"Name": "frontend"
			}],
			"Name": "pat-spring-doge-net-15",
			"Location": "West US"
		}
```
Then import the whole thing.
```shell
azure network import net-config-new.json
```

#### Creating VMs

Create the first VM in the vnet. Typically you would have an index i for your VM names, like my-vm-1. And increment that index as you add more machines. This time we specify a port for ssh, here 22001. This will automatically create an endpoint for that VM opening ssh on that port. Since all VMs are hidden behind the same cloud service, it is important to assign different ssh ports to each VM. Here I use core for username, the default CoreOS admin username, and specify the --no-ssh-password option so that we can only login with our private key. <vm-name> is what you would use for fields name and hostname in cloud-init.yml. Specify a string as the name of the availability set you want to create.

```shell
azure vm create \
<cloud-service-name> \
<image-name> \
core \
--vm-size Small \
--vm-name <vm-name> \
--availability-set <as-name> \
--location "West US" \
--ssh <ssh-port> \
--ssh-cert <Your pem file for ssh> \
--no-ssh-password \
--virtual-network-name <vnet-name> \
--subnet-names frontend \
--custom-data <cloud-init-file.yml>
```

Then for each subsequent VMs you want to add to the cluster, you need to add the --connect option, to connect the new VM to the existing ones in the cloud service, use a different ssh port (typically incrementing by 1 for each machine), and specify a different VM name (typically incrementing a number). This should be easy to automate and script. You need to create at least 3 VMs in your cluster, this is necessary for the Raft protocol used by etcd.
```shell
azure vm create \
<cloud-service-name> \
<image-name> \
core \
--connect \
--vm-size Small \
--vm-name <vm-name-i+1> \
--availability-set <as-name> \
--location "West US" \
--ssh <ssh-port-+1> \
--ssh-cert <Your pem file for ssh> \
--no-ssh-password \
--virtual-network-name <vnet-name> \
--subnet-names frontend \
--custom-data <cloud-init-file-i+1.yml>
```
### Creating load balanced endpoints for the service

All VMs will run a service that we want to load balance. Here I will use a java service that runs on port 8080 on each machine. In order to let Azure load balance these services, I create a load balance set called http for each VM, on port 80, directing to 8080 on each VM.
For each machine indexed i:
```shell
azure vm endpoint --lb-set-name http create <vm-name-i> 80 8080
```

### Checking the cluster

Same as previously,
```shell
ssh-add ~/.ssh/my_private_key.key
fleetctl --tunnel <cloud-service-name>.cloudapp.net:<ssh-port> list-machines
```
Here, you can use any of the ssh ports you have setup when creating VMs. A convenient way to work with fleetctl is to set FLEETCTL_TUNNEL in your ~/.bashrc.
```shell
export FLEETCTL_TUNNEL=<cloud-service-name>.cloudapp.net:<ssh-port>
```

## Deploying an application in your cluster

Many Enterprise Java developers are using the Spring Framework. I picked @joshlong & @phillip_webb's latest sample app, showcasing Spring Boot, a framework designed to build micro services, and Java 1.8, [Spring-doge](https://github.com/joshlong/spring-doge). It's a state of the art implementation of the [Doge meme](http://en.wikipedia.org/wiki/Doge_(meme) :-). You can watch Josh explain the [code behind Spring Doge on Youtube](https://www.youtube.com/watch?v=eCos5VTtZoI).

<a href="http://www.youtube.com/watch?feature=player_embedded&v=eCos5VTtZoI
" target="_blank"><img src="http://img.youtube.com/vi/eCos5VTtZoI/0.jpg"
alt="Josh on Spring-doge" width="240" height="180" border="10" /></a>

Spring-doge uses a MongoDB Database as a backend. You can setup a MongoDB cluster in your CoreOS cluster. To keep the example simple, I'd suggest using a hosted MongoDB: you can have one small instance of MongoDB for free using the [MongoLabs Azure Add-on](http://azure.microsoft.com/en-us/gallery/store/mongolab/mongolab/). Signup there and copy your Mongo connection uri: mongodb://username:password@hotname:port/dbname

@jamesdbloom built a convenient Docker container with Java 8 and Maven, jamesdbloom/docker-java8-maven.
I just extended his container to checkout Spring-doge, compile it, and run it, passing to it the MongoDB connection uri in environment variable MONGODB_URI. Here is the [Dockerfile for chanezon/spring-doge](../../../master/coreos/cloud-init/spring-doge/Dockerfile). I built that image at [chanezon/spring-doge](https://registry.hub.docker.com/u/chanezon/spring-doge/), but feel free to build and use your own.

```shell
FROM jamesdbloom/docker-java8-maven

MAINTAINER Patrick Chanezon <patrick@chanezon.com>

EXPOSE 8080

#checkout and build spring-doge
WORKDIR /local/git
RUN git clone https://github.com/joshlong/spring-doge.git
WORKDIR /local/git/spring-doge
RUN mvn package
CMD java -Dserver.port=8080 -Dspring.data.mongodb.uri=$MONGODB_URI -jar spring-doge/target/spring-doge.jar
```

CoreOS has a nice documentation about how to use [Fleet to launch Docker containers](https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/). Based on that, we will submit a fleet unit file for [spring-doge-http@.service](../../../master/coreos/cloud-init/spring-doge/spring-doge-http@.service), then deploy n instances of this unit, where n is the size of your cluster.

Here is the Fleet unit template file. You need to replace the MONGODB_URI variable in it before submitting it.

```shell
[Unit]
Description=spring-doge

[Service]
ExecStartPre=-/usr/bin/docker kill spring-doge-%i
ExecStartPre=-/usr/bin/docker rm spring-doge-%i
ExecStart=/usr/bin/docker run --rm --name spring-doge-%i -e MONGODB_URI=mongodb://username:password@hotname:port/dbname -p 8080:8080 chanezon/spring-doge
ExecStop=/usr/bin/docker stop spring-doge-%i

[X-Fleet]
Conflicts=spring-doge-http@*.service
```

And how to submit it to the cluster:

```shell
fleetctl submit spring-doge-http@.service
fleetctl list-unit-files
fleetctl start spring-doge-http@{1..3}.service
fleetctl list-units
UNIT MACHINE ACTIVE SUB
spring-doge-http@1.service 415162a4.../10.0.0.5 active running
spring-doge-http@2.service 7c7f60e0.../10.0.0.4 active running
```

In a browser navigate to http://cloud-service-name.cloudapp.net/, and have fun with Spring-doge!

Here's one of my [deployed version of it](http://pat-spring-doge-2.cloudapp.net/).

<img src="../../../master/coreos/cloud-init/spring-doge/spring-doge-simon.png"/>

## Future work

There are a few things I want to add to this tutorial in next few week:
* My first idea was to automate all that provisioning in a tool (generate the cloud-init files from a template). Instead of writing it, I looked around and found  [@kenperkins's coreos-cluster-cli](https://github.com/kenperkins/coreos-cluster-cli) to add azure support in it. It works on Rackspace. If it's extensible enough, I'll probably contribute to it instead of rolling my own tool.
* document a scalable architecture such as described in [CoreOS Cluster Architectures](https://coreos.com/docs/cluster-management/setup/cluster-architectures/)
* Deploying a replicated mongodb service in the cluster
* Describing how to configure docker to be able to connect to it remotely

## Another example: Netflix Eureka

@NetflixOSS [recently released Docker containers for many of their stack's components](http://techblog.netflix.com/2014/11/zerotodocker-easy-way-to-evaluate.html), with the [ZeroToDocker project](https://github.com/Netflix-Skunkworks/zerotodocker/wiki/Running-the-Images). This makes it very easy to get started playing with them.

I created an [example fleet service definition](../../../master/coreos/cloud-init/netflix-eureka/eureka-http@.service) for the Netflix [Eureka service registry](https://github.com/Netflix/eureka) server. Note that the Netflix instructions tell you to start the Docker container in detached mode: you have to remove that option, since [containers started in this mode don't work with Fleet](https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/). I map container port to 8081 on the host, since 8080 is already taken by spring-doge. This will allow us to deploy this service alogside spring-doge.

```shell
fleetctl submit netflix-eureka-http@.service
fleetctl list-unit-files
fleetctl start netflix-eureka-http@{1..3}.service
fleetctl list-units
UNIT MACHINE ACTIVE SUB
UNIT				MACHINE			ACTIVE	SUB
eureka-http@1.service		4830c4b8.../192.168.0.5	active	running
eureka-http@2.service		701e7304.../192.168.0.6	active	running
eureka-http@3.service		d00a6b6f.../192.168.0.4	active	running
spring-doge-http@1.service	4830c4b8.../192.168.0.5	active	running
spring-doge-http@2.service	701e7304.../192.168.0.6	active	running
spring-doge-http@3.service	d00a6b6f.../192.168.0.4	active	running
```
Then create a load balanced endpoint for each azure VM.

```shell
azure vm endpoint --lb-set-name eureka create <vm-name-i> 8080 8081
```
In a browser navigate to http://cloud-service-name.cloudapp.net:8080/eureka/, and have fun with Eureka!

<img src="../../../master/coreos/cloud-init/netflix-eureka/netflix-eureka.png"/>

## FAQ

I will document here answers to common questions:

### How do I connect from Docker cli on my dev machine to the Docker daemon on my CoreOS instance?

By default CoreOS is configured to have Docker listen only locally, so that you're supposed to use Fleet to manage your Docker containers. If you want to manage your containers directly, using Docker cli or Fig, you need to [customize your instance's Docker daemon settings](https://coreos.com/docs/launching-containers/building/customizing-docker/). From an Azure perspective, you need to think about opening an enpoint for the port you just opened, so that it's accessible from the outside.
