# MongoDB 4.4 docker build with LVM volumes for data

This code is meant to build a MongoDB 4.4 source environment with three shards and two replica sets each shard.
The MongoDB data volume is external and is managed with LVM to allow for volume growth and snapshoting.
The host for Docker of choice is Ubuntu 20.04 LTS but it can be any OS capable or running Docker and LVM.

Based on <https://www.linuxsysadmin.ml/2018/07/mongodb-sharding-con-docker.html>

## Creation of LVM volume

Create one LVM volume and mount in /mnt/mongodata

Use fdisk or parted to create a single primary partition on the whole volume.
The use LVM tools to create the LVM volume.

    pvcreate /dev/xvdf1  
    vgcreate vgmongodata /dev/xvdf1

We can create the volume smaller than the max capacity to test volume changes. In this case a 3GB volume from a 5GB block device.
Note he change of group ownership to the group *docker* to allow for non-root users running docker to bring the environment up.

    lvcreate --size 3G -n lvmongodata01 vgmongodata  
    mkfs.xfs /dev/vgmongodata/lvmongodata01

    mkdir /mnt/mongodata  
    mount /dev/vgmongodata/lvmongodata01 /mnt/mongodata/  
    chown root:docker /mnt/mongodata  
    chmod g+w /mnt/mongodata  

Add an entry to *fstab* so the volume is mounted automatically after booting the host.

    /dev/mapper/vgmongodata-lvmongodata01 /mnt/mongodata xfs defaults 0 0

As ubuntu or any other regular OS user in the docker OS group create data directories for each component of the environment.

    mkdir /mnt/mongodata/mongos01 /mnt/mongodata/config01 /mnt/mongodata/config02 /mnt/mongodata/config03 /mnt/mongodata/aquila01 /mnt/mongodata/aquila02 /mnt/mongodata/aquila03 /mnt/mongodata/lyra01 /mnt/mongodata/lyra02 /mnt/mongodata/lyra03 /mnt/mongodata/cygnus01 /mnt/mongodata/cygnus02 /mnt/mongodata/cygnus03

## Creating Docker images

Clone this repository to have the Docker files ready.

There are two Docker compose files in this repository:

* docker-compose.yml
* docker-compose-4.4.yml

The first one uses manually build images for each of the components. This allows for finer configuration control of each of the images.
These images can be built as follows

    docker build -t mongo-client build/mongo
    docker build -t mongo-server build/mongod
    docker build -t mongo-proxy build/mongos

Check the newly created docker images

    docker images
    REPOSITORY     TAG       IMAGE ID       CREATED          SIZE
    mongo-client   latest    8339a4f07f84   28 minutes ago   60.5MB
    mongo-proxy    latest    8339a4f07f84   28 minutes ago   60.5MB
    mongo-server   latest    8339a4f07f84   28 minutes ago   60.5MB

## Test the configuration so far with the Docker compose file of choice

    docker-compose up -d
or

    docker-compose --file docker-compose-4.4.yml up -d

The expected output looks like this. Note the PORTS column may vary if different ports are exposed or mapped.

    docker  ps

    CONTAINER ID   IMAGE       COMMAND                  CREATED             STATUS             PORTS       NAMES
    1be83edff266   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   lyra01
    ea1df4216bab   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   mongos01
    e35e31ee37c7   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   config02
    472373a31e13   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   lyra03
    7dbb21575c74   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   cygnus02
    7cc19be55847   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   aquila02
    059e31b895d5   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   cygnus01
    7031ad4bf8a7   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   config01
    ff15f9ebfa05   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   cygnus03
    81a9fdd88f1d   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   config03
    376eedb76ea8   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   lyra02
    39fe420e6263   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   aquila01
    62e55581158a   mongo:4.4   "docker-entrypoint.s…"   About an hour ago   Up About an hour   27017/tcp   aquila03

## Tie the cluster defining replica sets, config servers and shards

At this point our MongoDB servers are up and running but disconnected and without data.
We have to enable the replication and the sharding so in order to connect to the different MongoDB servers we will be using a disposable Docker image with a mongo client.
At this point, if the mongo-client image has not been built we can do it now.

    docker build -t mongo-client build/mongo

End now we can create a container and use it to connect to the different servers. Note the use of the default network name created by Docker compose. You may have to change this parameter in your environment to match the actual network name.

     docker run -ti --rm --net sharding_default mongo-client
    / $

Using the MongoDB client we proceed to create the replica sets and the shards.

    mongo --host config01 --port 27019
    rs.initiate()
    rs.add("config02:27019")
    rs.add("config03:27019")
    exit

    mongo --host aquila01 --port 27018
    rs.initiate()
    rs.add("aquila02:27018")
    rs.addArb("aquila03:27018")
    exit

    mongo --host lyra01 --port 27018
    rs.initiate()
    rs.add("lyra02:27018")
    rs.addArb("lyra03:27018")
    exit

    mongo --host cygnus01 --port 27018
    rs.initiate()
    rs.add("cygnus02:27018")
    rs.addArb("cygnus03:27018")
    exit

The cluster is now ready but it does not have any shards, we proceed now to add them.

    mongo --host mongos01 --port 27017
    sh.addShard("aquila/aquila01:27018")
    sh.addShard("aquila/lyra01:27018")
    sh.addShard("aquila/cygnus01:27018")

Once the process is complete this is how it looks like:

    sh.status()
    --- Sharding Status ---
    sharding version: {
          "_id" : 1,
          "minCompatibleVersion" : 5,
          "currentVersion" : 6,
          "clusterId" : ObjectId("620142ec360f7b46ea5d94f9")
    }
    shards:
          {  "_id" : "aquila",  "host" : "aquila/aquila01:27018,aquila02:27018,aquila03:27018",  "state" : 1 }
          {  "_id" : "cygnus",  "host" : "cygnus/cygnus01:27018,cygnus02:27018",  "state" : 1 }
          {  "_id" : "lyra",  "host" : "lyra/lyra01:27018,lyra02:27018,lyra03:27018",  "state" : 1 }
    active mongoses:
          "4.4.12" : 1
    autosplit:
          Currently enabled: yes
    balancer:
          Currently enabled:  yes
          Currently running:  no
          Failed balancer rounds in last 5 attempts:  0
          Migration Results for the last 24 hours:
                  No recent migrations
    databases:
          {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                  config.system.sessions
                          shard key: { "_id" : 1 }
                          unique: false
                          balancing: true
                          chunks:
                                  aquila  342
                                  cygnus  341
                                  lyra    341
                          too many chunks to print, use verbose if you want to force print

## Loading data into the cluster

I will be using <https://github.com/feliixx/mgodatagen> to generate data and load it into the cluster so I have some stuff to work with.
After downloading and decompressing the latest version available from the repo I will be copying it to the /mnt/mongodata directory and run my disposable client mounting it:

      docker run -ti --rm --net sharding_default --mount type=bind,source=/mnt/mongodata/generator,target=/mnt/mongodata/generator mongo-client

The tool requires a configuration file and we can find several in the repository. I chose <https://github.com/feliixx/mgodatagen/blob/master/datagen/generators/testdata/full-faker.json> for a fast start. This file must be copied to /mnt/mongodata/generator as well.

    docker run -ti --rm --net sharding_default --mount type=bind,source=/mnt/mongodata/generator,target=/mnt/mongodata/generator mongo-client
    / $ cd /mnt/mongodata/generator/
    /mnt/mongodata/generator $ ./mgodatagen --file=full-faker.json --host=aquila01 --port=27018
    connecting to mongodb://aquila01:27018
    MongoDB server version 4.4.12

    collection test_bson: done          [====================================================================] 100%

    +------------+-------+-----------------+------------+
    | COLLECTION | COUNT | AVG OBJECT SIZE |  INDEXES   |
    +------------+-------+-----------------+------------+
    | test_bson  |   467 |            3742 | _id_  4 kB |
    +------------+-------+-----------------+------------+

    run finished in 160ms

At this point we have our sharded cluster up and running with preserved data in a LVM volume and we are ready to start our testing.