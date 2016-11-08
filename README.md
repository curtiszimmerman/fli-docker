# fli-docker

[![Build Status](https://travis-ci.com/ClusterHQ/fli-docker.svg?token=q3MVd7yngMiAP2Yk1Rkp&branch=master)](https://travis-ci.com/ClusterHQ/fli-docker)

The `fli-docker` utility is designed to simplify the deployment of stateful applications inside Docker containers.

This is achieved through creation of a Flocker Hub Stateful Application Manifest (SAM) file (aka. "manifest"), which is used side by side with the Docker Compose file.
The SAM file is a YAML file that defines data volumes from [ClusterHQ](https://clusterhq.com)'s Flocker Hub,
synchronizes data snapshots locally, and maps them to Docker volumes in the Docker Compose file.

## How to install

1. Visit the [Releases](https://github.com/ClusterHQ/fli-docker/releases) page on github to download the binary.

2. Place the binary in your path, example `/usr/local/bin/fli-docker`.

3. Make the binary executable, `chmod +x /usr/local/bin/fli-docker`

4. Make sure dependecies such as Docker and [Fli](https://clusterhq.com/fli/introduction/g) are intalled.

### Pre-reqs

- Install `docker` see [here](https://docs.docker.com/engine/installation/)
- Install `fli`
  - You can do this by running `docker pull clusterhq/fli` or by learning how to install the binary from https://fli-docs.clusterhq.com/

### Optional

`fli-docker` uses [LibCompose](https://github.com/docker/libcompose), so it doesnt need the `docker-compose` cli tool directly, but you can install Docker Compose cli if you want to be able to use the `docker-compose` cli for manipulating your containers with that tool.

  - https://docs.docker.com/compose/install/

## Usage

To utilize the ClusterHQ `fli-docker` utility, examine the following command line arguments.

```
$ fli-docker --help
Usage:
  fli-docker version  [options]  (Get current tool version)
  fli-docker run      [options]  (Run with a manifest to pull and use snapshots for the compose app)
  fli-docker snapshot [options]  (Snapshot existing FlockerHub volumes used by the compose app)
  fli-docker stop     [options]  (Just like running a docker-compose stop)
  fli-docker destroy  [options]  (Just like running a docker-compose rm -f)
  fli-docker --help   (Get this help message)

  For help on a specific command, use: $ fli-docker <subcommand> --help
```

## How it works

`fli-docker` relies on `fli` and FlockerHub to push and pull snapshots of data around and itself is only an integration point with Docker Compose so that managing volumes and snapshots with Docker Compose and `fli` is even easier.

Like `docker-compose`, `fli-docker` relies on a YAML file, called the SAM (Stateful Application Manifest.)

## Stateful Application Manifest (SAM)

The Stateful Application Manifest (SAM) looks similar to a Docker Compose file, with a few key changes.

- The `flocker_hub` node references an `endpoint` and a valid `tokenfile` for FlockerHub authentication.
- The volumes are defined by name, same names in the docker compose YAML file, and each references a `snapshot` or `branch` and a `volumeset` which the `snapshot` or `branch` belongs to.

The `fli-docker` utility takes a `docker-compose.yml` file as input, and translates
volumes in the Docker Compose file to volumes backed by FlockerHub snapshots.

[Compose File Reference Link](https://docs.docker.com/compose/compose-file/#/volumes-volume-driver)

```
# Named volume
  - datavolume1:/var/lib/mysql
  - 'datavolume2:/var/lib/mysql'
  - /my/path1:/tmp/path1
  - '/my/path2':/tmp/path2
```

An example of a Stateful App Manifest (SAM) YAML file could be `dev-manifest.yml` below. Notice, under the `volumes:` section of the manifest, that each named volume references a `volumeset` and a `snapshot`.

You can obtain these identifiers from the FlockerHub user interface, or the `fli` command line utility.
Documentation about FlockerHub and `fli` itself can be found at [ClusterHQ Documentationn](https://clusterhq.com).

```yaml
docker_app: docker-compose-app.yml

flocker_hub:
    endpoint: https://data.flockerhub.clusterhq.com
    tokenfile: /root/fh.token

volumes:
    - name: redis-data
      snapshot: example-snapshot-1
      volumeset: docker-app-example
    - name: artifacts
      branch: example-artifacts-branch
      volumeset: docker-app-example
    - name: /my/path
      snapshot: example-snapshot-3
      volumeset: docker-app-example
```

The Docker Compose file (docker-compose-app.yml) that the SAM file leverages could be:

```yaml
version: '2'
services:
  web:
    image: clusterhq/moby-counter
    environment:
       - "USE_REDIS_HOST=redis"
    links:
      - redis
    ports:
      - "80:80"
    volumes:
      - artifacts:/myapp/artifacts/
  redis:
    image: redis:latest
    volumes:
       - 'redis-data:/data'
       - /my/path:/tmp/path
```

In this case, the CLI commands above would perform the necessary `pull` and `create`
commands with fli and manipulate the docker-compose file so that when the application is started it can be started with volumes based on your snapshots.

- `redis-data` would become snapshot: `example-snapshot-1`
- `artifacts` would become snapshot : `example-snapshot-2`
- `/my/path` would become snapshot: `example-snapshot-3`

### Using Branches

FlockerHub allows a user to create a volume from the tip of a branch which is
comprised of many volumes. This way, by referencing a branch instead of a volume
you will always get the tip snapshot of that branch.

#### To use a branch

Use `branch` instead of `snapshot` to get to latest snapshot in that branch.

```
volumes:
    - name: redis-data
      branch: branch-name
      volumeset: docker-app-example
```

What is looks like when using `fli-docker run`
```
$ fli-docker run -f fli-manifest.yml -c
MESSAGE: Parsing the fli manifest...
MESSAGE: Pulling FlockerHub volumes...
MESSAGE: Creating volume from branch...
MESSAGE: Mapping new volumes in compose file...
INFO[0034] [0/1] [appname]: Starting                    
INFO[0034] [1/1] [appname]: Started
```

## Example

You can use the example here in this repository. Follow the below instructions.

### fli-docker run

Use the example if you want.
```
$ git clone https://github.com/ClusterHQ/fli-docker/

$ cd fli-docker/examples/redis-moby
```

Run the example

> Note: you will need to set the `tokenfile` in the `fli-manifest.yml` or provide them as flags to `fli-docker run` using `-e https://flockerhub.com -t /root/your.token`.

> Note: you will need to create three snapshots and a volumeset to do this. See [this README](examples/redis-moby/README.md)

```
$ fli-docker run -f fli-manifest.yml -c -p myproject
MESSAGE: Parsing the fli manifest...
MESSAGE: Pulling FlockerHub volumes...
MESSAGE: Creating volumes from snapshots...
MESSAGE: Mapping new volumes in compose file...
INFO[0005] [0/2] [redis]: Starting                      
INFO[0005] [1/2] [redis]: Started                       
INFO[0005] [1/2] [web]: Starting                        
INFO[0006] [2/2] [web]: Started

$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                NAMES
dd97db0fc133        clusterhq/moby-counter:dcsea   "node index.js"          6 seconds ago       Up 5 seconds        0.0.0.0:80->80/tcp   myproject_web_1
5d535f5e1d55        redis:latest                   "docker-entrypoint.sh"   6 seconds ago       Up 5 seconds        6379/tcp             myproject_redis_1


$ docker inspect -f "{{.Mounts}}" redismoby_web_1
[{ /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/e4aed515-4f80-426c-ae13-b9a7b0487ab4 /myapp/artifacts  rw true rprivate}]

$ docker inspect -f "{{.Mounts}}" redismoby_redis_1
[{ /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/94ec5b24-1f3a-4695-b172-d17b840596c5 /data  rw true rprivate} { /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/c574874b-822e-4bf3-8a25-e63b4733619e /tmp/path  rw true rprivate}]
```

Optionally you dont have to specify `-c` or  `-p` so you can start the compose app yourself. Without `-c`
`fli-docker` will just modify the docker compose file and let you manage bring the services up.

```
$ fli-docker run -f fli-manifest.yml
MESSAGE: Parsing the fli manifest...
MESSAGE: Pulling FlockerHub volumes...
MESSAGE: Creating volumes from snapshots...
MESSAGE: Mapping new volumes in compose file...

$ cat docker-compose-app1.yml
version: '2'
services:
  web:
    image: clusterhq/moby-counter:dcsea
    environment:
       - "USE_REDIS_HOST=redis"
    links:
      - redis
    ports:
      - "80:80"
    volumes:
      - /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/e4aed515-4f80-426c-ae13-b9a7b0487ab4:/myapp/artifacts/
  redis:
    image: redis:latest
    volumes:
       - '/chq/907ee560-0110-4ab2-aaad-091ed9bb474f/94ec5b24-1f3a-4695-b172-d17b840596c5:/data'
       - /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/c574874b-822e-4bf3-8a25-e63b4733619e:/tmp/path

$ docker-compose -f docker-compose-app1.yml up -d
Pulling web (clusterhq/moby-counter:dcsea)...
dcsea: Pulling from clusterhq/moby-counter
a3ed95caeb02: Pull complete
93a86e942d51: Pull complete
faecfcc1d7ff: Pull complete
ddf3e3db435e: Pull complete
81a4604c1077: Pull complete
f8f4d4eabd85: Pull complete
Digest: sha256:e11ed56f5dad87ddef9865e758067ae6a182c234d9f10d1cf5c2a7d18a811eea
Status: Downloaded newer image for clusterhq/moby-counter:dcsea
Creating redismoby_redis_1
Creating redismoby_web_1

$ docker-compose -f docker-compose-app1.yml ps
      Name                     Command               State         Ports        
-------------------------------------------------------------------------------
redismoby_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp           
redismoby_web_1     node index.js                    Up      0.0.0.0:80->80/tcp 

$ docker inspect -f "{{.Mounts}}" redismoby_web_1
[{ /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/e4aed515-4f80-426c-ae13-b9a7b0487ab4 /myapp/artifacts  rw true rprivate}]

$ docker inspect -f "{{.Mounts}}" redismoby_redis_1
[{ /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/94ec5b24-1f3a-4695-b172-d17b840596c5 /data  rw true rprivate} { /chq/907ee560-0110-4ab2-aaad-091ed9bb474f/c574874b-822e-4bf3-8a25-e63b4733619e /tmp/path  rw true rprivate}]
```

### fli-docker stop

To stop your compose services you can run the following. Using the `example/redis-moby` in the below example.

```
$ cd fli-docker/examples/redis-moby
```

```
$ fli-docker stop -f fli-manifest.yml 
MESSAGE: Parsing the fli manifest...
INFO[0000] [0/2] [web]: Stopping                        
INFO[0000] [0/2] [redis]: Stopping                      
INFO[0000] [0/2] [redis]: Stoppe                       
INFO[0060] [0/2] [web]: Stopped  
```

### fli-docker destroy

To stop and force remove your containers, you can run the following. Using the `example/redis-moby` in the below example.

```
$ cd fli-docker/examples/redis-moby
```

```
$ fli-docker destroy -f fli-manifest.yml 
MESSAGE: Parsing the fli manifest...
INFO[0000] [0/2] [web]: Deleting                        
INFO[0000] [0/2] [redis]: Deleting                      
INFO[0000] [0/2] [redis]: Deleted                       
INFO[0000] [0/2] [web]: Deleted 
```

### fli-docker snapshot

Once you have a compose app running with `fli-docker`, you can snapshot and optionally push the volumes back to FlockerHub.

Snapshot the volumes and push them to FlockerHub
```
$ fli-docker snapshot -push
MESSAGE: Snapshotting and Pushing volumes to FlockerHub...
MESSAGE: Snapshotting and Pushing fli-1517ade4-502c-4781-b1e7-292b1cf329e7 from Volumeset docker-app-example
MESSAGE: Snapshotting and Pushing fli-5abb45f7-0fe6-4644-9042-3abda6a2b3a2 from Volumeset docker-app-example
MESSAGE: Snapshotting and Pushing fli-eef1035a-4105-4b92-902d-f33aa1dfa069 from Volumeset docker-app-example

```

Snapshot the volumes, but do not push them to FlockerHub
```
$ fli-docker snapshot
MESSAGE: Snapshotting volumes...
MESSAGE: Snapshotting fli-1517ade4-502c-4781-b1e7-292b1cf329e7 from Volumeset docker-app-example
MESSAGE: Snapshotting fli-5abb45f7-0fe6-4644-9042-3abda6a2b3a2 from Volumeset docker-app-example
MESSAGE: Snapshotting fli-eef1035a-4105-4b92-902d-f33aa1dfa069 from Volumeset docker-app-example
```

### Notes

- You may run this from anywhere `docker` and `fli` are installed.
- Snapshots would need to be pushed to volumesets in ClusterHQ Flocker Hub using a manifest that references them, otherwise `pull` will fail.
