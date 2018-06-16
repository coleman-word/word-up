# Documentation

# Cloudflare Proxy

Incoming container traffic can be optionally proxied through the Cloudflare servers. When operating in this mode Cloudflare provides HTTPS for container services, and it protects against distributed denial of service, customer data breach and malicious bot abuse.

To enable Cloudflare proxied traffic, it is sufficient to uncomment the following lines in the config.tfvars file, specifying DNS records to be proxied:

# Cloudflare proxy (optional)
cloudflare_proxied = "true"
cloudflare_record_texts = ["record1","record2",...] # Warn: wildcards are not supported when using proxied records

# Cloudflare DNS Records

Cloudflare runs one of the largest authoritative DNS networks in the world. In order to resolve domain names for exposed services, KubeNow can optionally configure the Cloudflare dynamic DNS service, so that a base domain name will resolve to the edge nodes (or the master node if no edge node is deployed).

To configure the Cloudflare dynamic DNS service, it is sufficient to uncomment the following lines in the config.tfvars file, specifying credentials, domain and subdomain:

# Cloudflare configuration (optional)
use_cloudflare = "true"
cloudflare_email = "your-cloudflare-email"
cloudflare_token = "your-cloudflare-token"
cloudflare_domain = "your-domain-name"
cloudflare_subdomain = "your-subdomain-name"

# Single-Node Deployments

When resources are scarce, or for testing purpose, KubeNow enables single-node deployments. In fact, it is possible to deploy the master node only, which will automatically enabled for service scheduling.

You can achieve this by commenting all of the lines for the other instance types (i.e. edge node, gluster node and worker node) in the config.tfvars file, leaving the master node only as it is shown below:

Master configuration (mandatory in general, above all for single-server setup)
master_flavor = "your-master-flavor"
master_as_edge = "true"

Node configuration
node_count = "3"
node_flavor = "your-node-flavor"

Edge configuration
edge_count = "2"
edge_flavor = "your-edge-flavor"

Gluster configuration
glusternode_count = "1"
glusternode_flavor = "your-glusternode-flavor"
glusternode_extra_disk_size = "200" # Size in GB

# GlusterFS Nodes

GlusterFS nodes are specialized service nodes. They run only GlusterFS and they are attached to a block storage volume to provide additional capacity. In the default settings, we don’t deploy GlusterFS nodes, as it is not required in many use cases. However, GlusterFS can be particularly convenient when a distributed file system is needed for container synchronization.

To deploy GlusterFS nodes, it is sufficient to uncomment the following lines in the config.tfvars file, and to set the desired number of edge nodes, along with an available instance flavor and block storage disk size:

## Gluster configuration
glusternode_count = "1"
glusternode_flavor = "your-glusternode-flavor"
glusternode_extra_disk_size = "200" # Size in GB

# Edge Nodes

Edge nodes are specialized service nodes with an associated public IP address, and they run Traefik acting as reverse proxies, and load balancers, for the services that are exposed to the Internet. In the default settings, we don’t deploy any edge node enabling the reverse proxy logic in the master node instead. However, in production settings we recommend to deploy one or more edge nodes to reduce the load in the master.

To deploy edge nodes, it is sufficient to uncomment the following lines in the config.tfvars file, and to set the desired number of edge nodes, along with an available instance flavor:

## Master configuration (mandatory in general, above all for single-server setup)
master_flavor = "your-master-flavor"
master_as_edge = "false"

## Edge configuration
edge_count = "2"
edge_flavor = "your-edge-flavor"
Please notice that we set master_as_edge = "false" to disable Traefik in the master node.

# Deploy on Amazon Web Services (AWS)

## Prerequisites

In this section we assume that:

You have an IAM user along with its access key and security credentials (http://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html)
Deployment configuration

First we need to initialize a deploy configuration directory by running:

kn init aws my_deployment
The configuration directory contains a new SSH key pair for your deployments, and a Terraform configuration template that we need to fill in.

Locate into my_deployment :

cd my_deployment
In the configuration file config.tfvars you will need to set at least:

## Cluster configuration

cluster_prefix: every resource in your tenancy will be named with this prefix
aws_region: the region where your cluster will be bootstrapped (e.g. eu-west-1)
availability_zone: an availability zone for your cluster (e.g. eu-west-1a)
Credentials

aws_access_key_id: your access key id
aws_secret_access_key: your secret access key
Master configuration

master_instance_type: an instance type for the master (e.g. t2.medium)
master_disk_size: edges disk size in GB
Node configuration

node_count: number of Kubernetes nodes to be created
node_instance_type: an instance type for the Kubernetes nodes (e.g. t2.medium)
node_disk_size: edges disk size in GB
Deploy KubeNow

Once you are done with your settings you are ready deploy your cluster running:

kn apply
To check that your cluster is up and running you can run:

kn kubectl get nodes
As long as you are in the my_deployment directory you can use kubectl over SSH to control Kubernetes. If you want to open an interactive SSH terminal onto the master then you can use the kn ssh command:

kn ssh
If everything went well, now you are ready to deploy your first application.

# Deploy your first application

In this guide we are going to deploy a simple application: cheese. We will deploy 3 web pages with a 2 replication factor. The master node will act as a reverse proxy, load balancing the requests among the replicas in the Kubernetes nodes.

The simple cluster that we just deployed uses nip.io as base domain for incoming HTTP traffic. First, we need to figure out our cluster domain by running:

grep domain inventory
The command will return something like domain=37.153.138.137.nip.io, meaning that our cluster domain name in this case would be 37.153.138.137.nip.io.

In KubeNow we encourage to deploy and define SaaS-layer applications using Helm. The KubeNow community maintain a Helm repository that contains applications that are developed and tested for KubeNow: https://github.com/kubenow/helm-charts. To deploy the cheese application you can run the following command, substituting <your-domain> with the domain name that you got above:

kn helm install --name cheese --set domain=<your-domain> kubenow/cheese
If everything goes well you should be able to access the web pages at:

http://stilton.<your-domain>
http://cheddar.<your-domain>
http://wensleydale.<your-domain>
Traefik reverse proxy

KubeNow uses the Traefik reverse proxy as ingress controller for your cluster. Traefik is installed on one or more nodes, namely edge nodes, which have a public IP associated. In this way, we can access services with a few floating IP without needing LBaaS, which may not be available on certain cloud providers.

In the default setting KubeNow doesn’t deploy any edge node, and it runs Traefik in the master node.

Accessing the Traefik UI

One simple and quick way to access the Traefik UI is to tunnel via SSH to one of the edge nodes with the following command:

ssh -N -f -L localhost:8081:localhost:8081 ubuntu@<your-domain>
Using SSH tunnelling, the Traefik UI should be reachable at http://localhost:8081, and it should look something like this:

../_images/traefik_UI_example.png
In the left side you can find your deployed frontends URLs, whereas on the right side the backend services.

# Clean after yourself

Cloud resources are typically pay-per-use, hence it is good to release them when they are not used. Here we show how to destroy a KubeNow cluster.

To release the resources, please run:

kn destroy
Warning: if you delete the cluster configuration directory (my_deployment) the cluster status will be lost, and you’ll have to delete the resources manually.

