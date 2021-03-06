# dockerized-chef-services
Docker definitions for Chef Server and Automate 1.x

![Architecture Diagram](https://www.lucidchart.com/publicSegments/view/4f01dc86-c34a-49a9-b619-f3d1056e7a41/image.png)

## Release Notes
Please read the release notes [here](https://github.com/chef-customers/dockerized-chef-services/blob/master/RELEASE_NOTES.md) for any notable changes.

## How it works

* All of the Chef Server and Chef Automate services have been packaged as Habitat `.hart` files
* The hab packages have been exported to docker containers and published to docker hub
* These containers support running under a random, arbitrary effective user and/or group id
* Launch containers via the included [docker-chef.sh](https://github.com/chef-customers/dockerized-chef-services/tree/master/terraform/docker-chef.sh) script.
* Each host forms its own non-permanent Habitat gossip ring, sharing service discovery data intra-host only
* Environment variables are used to configure settings from the default

## Deploying it

### Using AWS and Terraform

* Customize the `terraform.tfvars.example` [Terraform](https://terraform.io) configuration file, rename it to `terraform.tfvars` and deploy with `terraform apply`
* Sit back and enjoy - in about 4 minutes you'll have a cluster up and running

```bash
git clone https://github.com/chef-customers/dockerized-chef-services.git
cd dockerized-chef-services/terraform
cp terraform.tfvars.example terraform.tfvars
vi terraform.tfvars # change anything that needs customization
terraform init
terraform apply
```

### Manual

* Provision two hosts as depicted in the Architecture diagram above and ensure the items from [Deployment notes](https://github.com/chef-customers/dockerized-chef-services#deployment-notes)
* Copy `docker-run.sh` to each host
* Create a customized `env.sh` and place in the same directory as `docker-run.sh`
* Create the `$DATA_MOUNT` mount point on each host and ensure that it's owned by `$USER_ID:$GROUP_ID`
* Copy a `passwd` and `group` file (see examples below) into `$DATA_MOUNT` and ensure they're readable by `$USER_ID:$GROUP_ID`
* Run `docker-run.sh -h` to see the help menu

```
$ ./docker-chef.sh -h
This is a control script for starting|stopping Chef Server and Chef Automate docker services.

You must specify the following options:
 -s [automate|chef-server]           REQUIRED: Services type: Chef Server or Chef Automate
 -a [stop|start]                     REQUIRED: Action type: start or stop services
 -n [container name]                 OPTIONAL: The docker container name. Leaving blank implies ALL
 -h                                  OPTIONAL: Print this help message

 ex. ./docker-chef.sh -s chef-server -a start            # starts up all Chef Server services
 ex. ./docker-chef.sh -s automate -a stop -n logstash    # stops Automate's logstash service

$
```

Example `group`

```
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
systemd-journal:x:23:
input:x:24:
mail:x:34:
nogroup:x:99:
users:x:999:
hab:x:42:hab:
chef-dev-ux:x:8888:
```

Example `passwd`

```
root:x:0:0:root:/root:/bin/sh
bin:x:1:1:bin:/dev/null:/bin/false
daemon:x:6:6:Daemon User:/dev/null:/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/bin/false
hab:x:42:42:hab User:/:/bin/false
chef-dev-ux:x:9999:8888:Test User:/:/bin/false
```

Example `env.sh`:

```bash
#!/bin/bash

export LC_ALL=en_US.UTF-8

export USER_ID=9999
export GROUP_ID=8888
export DATA_MOUNT=/mnt/data
export ENTERPRISE=brewinc
export ADMIN_PASSWORD=insecurepassword
export HOST_IP=$(hostname --ip-address)
export AUTOMATE_ENABLED=true
export AUTOMATE_SERVER=172.31.20.110
export AUTOMATE_TOKEN=93a49a4f2482c64126f7b6015e6b0f30284287ee4054ff8807fb63d9cbd1c506
export CHEF_SERVER_DOCKER_ORIGIN=chefserverofficial
export CHEF_SERVER_VERSION=stable
export AUTOMATE_DOCKER_ORIGIN=chefdemo
export AUTOMATE_VERSION=stable
export DOCKER_REQUIRES_SUDO=false
export DOCKER_DETACH_CONTAINER=true
```

### Docker-compose

```bash
git clone https://github.com/chef-customers/dockerized-chef-services.git
cd dockerized-chef-services/docker
docker-compose -f chef-server.yml down && docker system prune --volumes -f && docker-compose -f chef-server.yml up
docker-compose -f automate.yml down && docker system prune --volumes -f && docker-compose -f automate.yml up
```

## Deployment notes

Some environments may require the following, particularly if Elasticsearch refuses to start:

```
sudo cat > /etc/sysctl.d/00-chef.conf <<EOF
vm.swappiness=10
vm.max_map_count=262144
vm.dirty_ratio=20
vm.dirty_background_ratio=30
vm.dirty_expire_centisecs=30000
EOF
sudo sysctl -p /etc/sysctl.d/00-chef.conf

sudo cat > /etc/security/limits.d/20-nproc.conf<<EOF
*   soft  nproc     65536
*   hard  nproc     65536
*   soft  nofile    1048576
*   hard  nofile    1048576
EOF
```

Ensuring that locale is UTF-8 is necessary if not already set:

```
export LC_ALL=en_US.UTF-8
```

For the non-root images, all the services run on non-privileged ports. To enable seamless external access
to TCP ports 80 and 443 the following iptables rules must be applied.
Alternatively, use a Load Balancer.

If not using a LB, on the Chef Server Host run:

```
sudo iptables -A PREROUTING -t nat -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -A PREROUTING -t nat -p tcp --dport 443 -j REDIRECT --to-port 8443
sudo iptables -t nat -A OUTPUT -o lo -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -t nat -A OUTPUT -o lo -p tcp --dport 443 -j REDIRECT --to-port 8443
```

And on the Automate Host:

```
sudo iptables -A PREROUTING -t nat -p tcp --dport 80 -j REDIRECT --to-port 8080
sudo iptables -A PREROUTING -t nat -p tcp --dport 443 -j REDIRECT --to-port 8443
```

# Operating it

## Starting | Stopping

For this use the `docker-chef.sh` script as shown above.

## Administrative Functions

### Chef Server

Run the functional test suite to ensure Chef server is working:

```
docker exec -it chef-server-ctl chef-server-test
```

Add users, orgs, etc to the Chef server

docker run:

```
docker exec -it chef-server-ctl chef-server-ctl (subcommands)
```

### Chef Automate

Coming Soon! We will be releasing an `automat-ctl` command in the `workflow-server` container.

## Logging
All container logs are directed to STDOUT. You should employ a Docker logging mechanism to ensure those logs are captured and aggregated in a central location. Being able to provide full logs from all containers is necessary in order to recieve support.

To view the status of any container:

docker run:

```
docker logs <container name or id>
```

An easy way to check on the health of Chef Server and Automate is to look for the health checks from `oc_erchef` and `workflow-server` containers accordingly. They will look like:

```
oc_erchef.default hook[health_check]:(HK): {"status":"pong","upstreams":{"chef_elasticsearch":"pong","chef_sql":"pong","oc_chef_authz":"pong"},
"keygen":{"keys":10,"max":10,"max_workers":10,"cur_max_workers":10,"inflight":0,"avail_workers":10,"start_size":0},"indexing":{"mode":"batch"}}
```

```
workflow-server.default hook[health_check]:(HK): {"status":"pong","configuration_mode":"standalone","fips_mode":"false","upstreams":
[{"postgres":{"status":"pong"},"lsyncd":{"status":"not_running"},"elasticsearch":{"status":"pong","cluster_name":"elasticsearch","status":
"yellow","timed_out":false,"number_of_nodes":1,"number_of_data_nodes":1,"active_primary_shards":20,"active_shards":20,"relocating_shards":0,
"initializing_shards":0,"unassigned_shards":20,"delayed_unassigned_shards":0,"number_of_pending_tasks":0,"number_of_in_flight_fetch":0,
"task_max_waiting_in_queue_millis":0,"active_shards_percent_as_number":50.0},"rabbitmq":{"status":"pong","vhost_aliveness":{"status":
"pong"},"node_health":{"status":"pong"}}}]}
```
