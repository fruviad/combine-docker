# Combine-Docker


## Overview

This repository provides a "Dockerized" version of [Combine](https://github.com/mi-dpla/combine.git). Combine is a Django application to facilitate the harvesting, transformation, analysis, and publishing of metadata records by Service Hubs for inclusion in the [Digital Public Library of America (DPLA)](https://dp.la/).

### How does it work?

Major components that support Combine have been broken out into distinct Docker images and containers.  Using [Docker Compose](https://docs.docker.com/compose/), each of these major components is associated with a Docker Compose **service**.  Some share base images, others are pulled from 3rd party Docker images (like ElasticSearch and Mongo).

Docker Compose provides a way to interact with all the containers that support Combine at once, even providing some improved ability to view logs, restart services, etc.

### Data Integrity Warning

The containerization of Combine provides arguably easier deployment and upgrading, but introduces some risks to data integrity in Combine.  Combine-Docker stores data that needs to persist between container rebuilding and upgrades in named volumes, specifically the following:

  * `esdata`: ElasticSearch data
  * `mongodata`: Mongo data
  * `mysqldata`: MySQL data
  * `hdfs`: Hadoop HDFS data
  * `combine_home`: Home directory for Combine that contains important, non-volatile data

Containers are shutdown with the command `docker-compose down`, which is perfectly safe and encouraged!  However, **do not** include the `-v` or `--volumes` flag to that command, which will remove **all** volumes associated with the services in the `docker-compose.yml` file.

Docker does a relatively good job protecting named volumes, but this simple command would wipe data from Combine.  You can find [more information about the command `docker-compose down` here](https://docs.docker.com/compose/reference/down/).

### Security Warning

Combine code should be run behind your institution's firewall on a secured server. Access to combine should be protected by your institution-wide identity and password system, preferably using two-factor authentication. If your institution supports using VPNs for access to the server's network that is a good additional step. 

This is in addition to Combine's own passwords. While we don't yet have explicit documentation on how to set up SSL inside the provided nginx in Combine, it's possible and strongly recommended to do so.

### Version Change Log for v0.11.1
[V0.11.1 Change Log](https://github.com/MI-DPLA/combine-docker/blob/master/combine_version_change_log.pdf)


## Installation and First Build

### Downloading combine-docker

**Windows only: An important git config before you download code!**

Before you clone the repository on Windows, ensure that you have your git configured not to add Windows-style line endings. I believe you can do this by setting:
```
git config --global core.autocrlf false
```

**Clone combine-docker**

The first install step is to clone this repository and move into it:

`git clone https://github.com/mi-dpla/combine-docker.git`  
`cd combine-docker`

### Initializing combine-docker
The complete instructions include important information on upgrading an existing Combine server. Using the detailed instructions is strongly recommended.

[Complete detailed Linux instructions](https://github.com/MI-DPLA/combine-docker/blob/master/combine_docker_detailed_installation_instructions.pdf)


**Abbreviated Instructions**

NOTE: All of the scripts assume you are building on Ubuntu 18.04 LTS.

**Windows only: In the next step run build.ps1 instead of build.sh**

Next, run the `build.sh` script:
```
./build.sh
```

**Note:** This script may take some time, anywhere from 5-20 minutes depending on your hardware.  This script accomplishes a few things:

  * initializes Combine Django app as Git submodule at `./combine/combine`
  * builds all required Docker images
  * runs one-time database initializations and migrations

## Configuration

Once a build is complete, configurations may be performed on Combine's `localsettings.py`.  This file is found at `./combine/combine/combine/localsettings.py`.  This file will be maintained between upgrades.


## Running and Managing

Ensuring that `first_build.sh` (or `update_build.sh` if appropriate) has been run, fire up all containers with the following:
```
docker-compose up -d
```

Logs can be viewed with the `logs` command, again, selecting all services, or a subset of:

**tail all logs:** `docker-compose logs -f`

**tail logs of specific services:** `docker-compose logs -f combine-django combine-celery livy`


As outlined in the [Combine-Docker Containers](#docker-images-and-containers) section all services, or a subset of, can be restarted as follows:

**To restart Combine Django app, background tasks Celery, and Livy:** `docker-compose restart combine-django combine-celery`

**To restart everything:** `docker-compose restart`


**To stop all services and containers** 
(**Reminder:** Do not include -v or --volumes flags, as these will wipe ALL data from Combine) Just use:`docker-compose down`


**To view stats of containers:** `docker stats`

## Updating

This Dockerized version of Combine supports, arguably, easier version updating because major components, broken out as images and containers, can be readily rebuilt.  Much like the repository Combine-Playbook, this repository follows the same versioning as Combine.  So checking out the tagged release `v0.11.1` for this repository, will build Combine version `v0.11.1`.

To update, follow these steps from the Combine-Docker repository root folder:

```
# pull new changes to Combine-Docker repository
git pull

# checkout desired release, e.g. v0.11.1
git checkout v0.11.1

# run update build script
./update_build.sh

# Restart as per normal
docker-compose up -d
```


## Docker Services and Volumes & Binds

This Dockerized version of Combine includes the following services, where each becomes a single container:

| Service Name          | Notes                                                      |
| --------------------- | ---------------------------------------------------------- |
| **host machine**      | not a container, but part of internal network              |
| `elasticsearch`       |                                                            |
| `mongo`               |                                                            |
| `mysql`               |                                                            |
| `redis`               |                                                            |
| `hadoop-namenode`     |                                                            |
| `hadoop-datanode`     |                                                            |
| `spark-master`        | not currently used                                         |
| `spark-worker`        | not currently used                                         |
| `combine-django`      |                                                            |
| `livy`                | location of spark application running in `local[*]` mode   |
| `combine-celery`      |                                                            |


The following tables show Docker volumes and binds that are created to support data sharing between containers, and "long-term" data storage.  The column `Data Storage` indicates which volumes act as data stores for Combine and should not be deleted (unless, of course, a fresh installation is desired).  Conversely, the column `Refreshed on Upgrade` shows which tables are removed during builds.  **Note:** this information is purely for informational purposes only; the build scripts and normal usage of `docker-compose up` and `docker-compose down` will not remove these volumes.


|Volume Name|Type|Source|Target|Data Storage|RefreshedonUpdate|AssociatedServices|
|-----------|----|------|------|------------|-----------------|------------------|
|`esdata`|namedvolume|n/a|`/usr/share/elasticsearch/data`|TRUE||elasticsearch|
|`mongodata`|namedvolume|n/a|`/data/db`|TRUE||mongo|
|`mysqldata`|namedvolume|n/a|`/var/lib/mysql`|TRUE||mysql|
|`hdfs`|namedvolume|n/a|`/hdfs`|TRUE||hadoop-namenode,hadoop-datanode|
|`combine_home`|namedvolume|n/a|`/home/combine`|TRUE||[spark-cluster-base]|
|`combine_django_app`|bind|`./combine/combine`|`/opt/combine`||TRUE|combine-django,combine-celery,livy|
|`combine_python_env`|namedvolume|n/a|`/opt/conda/envs/combine`||TRUE|combine-django,combine-celery,livy|
|`hadoop_binaries`|namedvolume|n/a|`/opt/hadoop`||TRUE|[spark-cluster-base]|
|`spark_binaries`|namedvolume|n/a|`/opt/spark`||TRUE|[spark-cluster-base]|
|`livy_binaries`|namedvolume|n/a|`/opt/livy`||TRUE|[spark-cluster-base]|
|`combine_tmp`|namedvolume|n/a|`/tmp`||TRUE|[spark-cluster-base]|


## Troubleshooting

### ElasticSearch container dies because of `vm.max_map_count`

Depending on machine and OS (Linux, Mac, Windows), might need to bump `vm.max_map_count` on Docker host machine (seems to be particularly true on older ones):
[https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-cli-run-prod-mode)

### Port collision error: `port is already allocated`

By default, nearly all relevant ports are exposed from the containers that conspire to run Combine, but these can turned off selectively (or changed) if you have services running on your host that conflict.  Look for the `ports` section for each service in the `docker-compose.yml` to enable or disable them.

### java.lang.ClassNotFoundException: org.elasticsearch.hadoop.mr.LinkedMapWritable

Make sure that the `elasticsearch-hadoop-x.y.z.jar` in `combinelib` matches the version specified in the `ELASTICSEARCH_HADOOP_CONNECTOR_VERSION` environment variable configured in your `.env`.

### Other issues?

Help is available for combine installation at `combine-support@umich.edu`. Also, please don't hesitate to [submit an issue](https://github.com/MI-DPLA/combine-docker/issues)!


## Development

The Combine Django application, where most developments efforts are targeted, is a [bind mount volume](https://docs.docker.com/storage/bind-mounts/) from the location of this cloned repository on disk at `./combine/combine`.  Though the application is copied to the docker images during build, to support the installation of dependencies, the location `/opt/combine` is overwritten by this bind volume at `docker-compose up` or `run`.  This allows live editing of the local folder `./combine/combine`, which is updating the folder `/opt/combine` in services `combine-django`, `combine-celery`, and `livy`.

The folder `./combine/combine` can, for the most part, be treated like a normal GitHub repository.  For example, one could checkout or create a new branch, and then push and pull from there.

## Automated Testing

Combine itself has automated tests. If you want to run them from inside here, you will need to uncomment the `ports` sections for mysql and mongo in `docker-compose.yml`, and you will also need to edit your /etc/hosts file to redirect `mysql` and `mongo` to `127.0.0.1`. This is because the host machine needs to have access to the databases for the Django test runner to set up and tear down around each run.
