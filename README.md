# MongoDB 4.4 docker build with LVM volumes for data

This code is meant to build a MongoDB 4.4 source environment with three shards and two replica sets each shard.
The MongoDB data volume is external and is managed with LVM to allow for volume growth and snapshoting.
The host for Docker of choice is Ubuntu 20.04 LTS but it can be any OS capable or running Docker and LVM.

Based on <https://www.linuxsysadmin.ml/2018/07/mongodb-sharding-con-docker.html>

## Creation of LVM volume

Create one LVM volume and mount in /mnt/mongodata

Use fdisk or parted to create a single primary partition on the whole volume.
The use LVM tools to create the LVM volume.

`pvcreate /dev/xvdf1`  
`vgcreate vgmongodata /dev/xvdf1`

We can create the volume smaller than the max capacity to test volume changes. In this case a 3GB volume from a 5GB block device.
Note he change of group ownership to the group *docker* to allow for non-root users running docker to bring the environment up.

`lvcreate --size 3G -n lvmongodata01 vgmongodata`  
`mkfs.xfs /dev/vgmongodata/lvmongodata01`

`mkdir /mnt/mongodata  
mount /dev/vgmongodata/lvmongodata01 /mnt/mongodata/  
chown root:docker /mnt/mongodata  
chmod g+w /mnt/mongodata  
`

Add an entry to *fstab* so the volume is mounted automatically after booting the host.

`/dev/mapper/vgmongodata-lvmongodata01 /mnt/mongodata xfs defaults 0 0`

As ubuntu or any other regular OS user in the docker OS group create data directories for each component of the environment.

`mkdir /mnt/mongodata/mongos01 /mnt/mongodata/config01 /mnt/mongodata/config02 /mnt/mongodata/config03 /mnt/mongodata/aquila01 /mnt/mongodata/aquila02 /mnt/mongodata/aquila03 /mnt/mongodata/lyra01 /mnt/mongodata/lyra02 /mnt/mongodata/lyra03 /mnt/mongodata/cygnus01 /mnt/mongodata/cygnus02 /mnt/mongodata/cygnus03`

## Creating Docker images

Clone this repository to have the Docker files ready.

There are two Docker compose files in this repository:

* docker-compose.yml
* docker-compose-4.4.yml

The first one uses manually build images for each of the components. This allows for finer configuration control of each of the images.
These images can be built as follows

`docker build -t mongo-client build/mongo
docker build -t mongo-server build/mongod
docker build -t mongo-proxy build/mongos
`

Check the newly created docker images

docker images
REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
mongo-client   latest    8339a4f07f84   28 minutes ago   60.5MB
mongo-proxy    latest    8339a4f07f84   28 minutes ago   60.5MB
mongo-server   latest    8339a4f07f84   28 minutes ago   60.5MB


# Configuration file contents

 ***************  Contents of mongod_aquila.conf **************

processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27018
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: aquila

sharding:
  clusterRole: shardsvr

 ***************  Contents of mongod_config.conf **************

processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27019
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: config

sharding:
  clusterRole: configsvr

 ***************  Contents of mongod_cygnus.conf **************

processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27018
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: cygnus

sharding:
  clusterRole: shardsvr

 ***************  Contents of mongod_lyra.conf **************

processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27018
  unixDomainSocket:
    enabled: false

storage:
  dbPath: /srv/mongodb
  engine: wiredTiger
  journal:
    enabled: true

replication:
  replSetName: lyra

sharding:
  clusterRole: shardsvr

 ***************  Contents of mongos.conf **************

processManagement:
  fork: false

net:
  bindIp: 0.0.0.0
  port: 27017
  unixDomainSocket:
    enabled: false

sharding:
   configDB: config/config01:27019,config02:27019,config03:27019

# Contents of docker-compose.yml

# Note that this version of the file does not include volume mappings to the LVM volume created above. This is to validate that everything works as expected

version: '3'
services:
  mongos01:
    image: mongo-proxy
    container_name: mongos01
    hostname: mongos01
    volumes:
      - ./mongos.conf:/etc/mongos.conf:ro
    restart: always
  config01:
    image: mongo-server
    container_name: config01
    hostname: config01
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  config02:
    image: mongo-server
    container_name: config02
    hostname: config02
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  config03:
    image: mongo-server
    container_name: config03
    hostname: config03
    volumes:
      - ./mongod_config.conf:/etc/mongod.conf:ro
    restart: always
  aquila01:
    image: mongo-server
    container_name: aquila01
    hostname: aquila01
    volumes:
      - ./mongod_aquila.conf:/etc/mongod.conf:ro
    restart: always
  aquila02:
    image: mongo-server
    container_name: aquila02
    hostname: aquila02
    volumes:
      - ./mongod_aquila.conf:/etc/mongod.conf:ro
    restart: always
  aquila03:
    image: mongo-server
    container_name: aquila03
    hostname: aquila03
    volumes:
      - ./mongod_aquila.conf:/etc/mongod.conf:ro
    restart: always
  lyra01:
    image: mongo-server
    container_name: lyra01
    hostname: lyra01
    volumes:
      - ./mongod_lyra.conf:/etc/mongod.conf:ro
    restart: always
  lyra02:
    image: mongo-server
    container_name: lyra02
    hostname: lyra02
    volumes:
      - ./mongod_lyra.conf:/etc/mongod.conf:ro
    restart: always
  lyra03:
    image: mongo-server
    container_name: lyra03
    hostname: lyra03
    volumes:
      - ./mongod_lyra.conf:/etc/mongod.conf:ro
    restart: always
  cygnus01:
    image: mongo-server
    container_name: cygnus01
    hostname: cygnus01
    volumes:
      - ./mongod_cygnus.conf:/etc/mongod.conf:ro
    restart: always
  cygnus02:
    image: mongo-server
    container_name: cygnus02
    hostname: cygnus02
    volumes:
      - ./mongod_cygnus.conf:/etc/mongod.conf:ro
    restart: always
  cygnus03:
    image: mongo-server
    container_name: cygnus03
    hostname: cygnus03
    volumes:
      - ./mongod_cygnus.conf:/etc/mongod.conf:ro
    restart: always

# Test the configuration so far

docker-compose up -d

# Tie the cluster defining replica sets, config servers and shards

# Bring up a container with a disposable mongo client

 docker run -ti --rm --net sharding_default mongo-client
