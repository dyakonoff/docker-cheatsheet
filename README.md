# docker cheatsheet

## CLI interface
A list of commands and hints for docker usage from

To check the version of docker
```
docker version
```

To check the config values of docker:
```
docker info
```

Docker command structure
* old style: `docker <command> (option)`
* new style: `docker <command> <sub-comand> (options)`

List containers:
```
docker container ls -a
docker ps -a
```

`docker container run` - starts a new container (or `docker run` in the old way)
`docker container start` - starts an existing container

most common cases:
* `docker container run --publish <container-port>:<host-port> --detach --name <container-local-name> <image-name> -T`
  
Use:
* option -e to pass environment variables into container: 

```bash
$ docker container run -p 3306:3306 -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql
```

`docker run container -P ...` - run a container and opens all ports that are mentioned in `EXPOSE` command.

* option `--rm` automatically removes the container whet it exits
```bash
$ docker container ru --rm -it --name centos centos:7 bash
```


* `docker container stop <id>` - stops the container
* `docker container rm <id>` - removes the container (or `docker rm` in the old way). USe `-f` flag to remove running containers.

Check the logs of the container
`docker container logs <container-name>`

`docker container top <container-name>` - top of the processes in the container

### Process monitoring inside containers

* `docker container top <container-id>` - process list in one container
* `docker container inspect` - details of one container config
* `docker container stats` - performance stats for **all** containers (or just one if container id is provided) in real time

### Getting shell inside containers

Start new container interactively:
`docker container run -it` - run nre container interactively
`docker container exec -it` - run additional command in **existing** container

Run an alternative command instead the default one when container starts:

```
$ docker container run --it --name proxy nginx bash
```

**Note:** that bash will start but nginx server will not, since the default command was replaced.
Container will be stopped when user exited bash.
This option is handy when you need to install additional packages into container or do other administrative stuff with it.

Attach to existing container (but not running) with a cli terminal:

```bash
$ docker container start -ai <container-name/id>
```

Connect to a running container and run additional process (bash/sh in our case):

```bash
$ docker container exec -it mysql bash
$ docker container exec -it nginx sh
```

## Networking

`docker container -p <host_port>:<container_port>` - port forwarding

`docker container port <container>` - quick ports check of the container.

All containers on a virtual (docketr) network can talk to each other without `-p` flag.
Best practice is to create a new virtual network for each app. Container can be connected to any number of virtual networks from **zero** to **N**.

Also, it's possible to skip virtual networks and use host IP (`--net=host`).

`--format` flag is a replacement for external `grep` command. `--format` is cleaner and more consistent then grep.

Example of how to get the IP address of the container:

```bash
$ docker container run -p 80:80 --name webhost -d nginx
ac58cfa521a22de438f6ea328271ef759f43ec0ee0e6b29153b8bc48b597840f
$ docker container inspect --format '{{ .NetworkSettings.IPAddress }}' webhost
172.17.0.2
$
```

Default docker network is: bridge/docker0.

Containers in the same virtual network can freely talk to each other even if they don't expose ports to host with `-p` or `--publish` flag.

Remember, that only **one** container can listen the port on a host level. 
In other words, a host port can't bil listened by more than one container at the same time.

* `docker network ls` - show networks
* `docker network inspect <network>` - inspect a network
* `docker network create <network_name>`
*  or `docker network create <network_name> --driver <driver_name>` create a network
* `docker network xonnect` - connect a network to container
* `docker network disconnect` - detach a network from container

Examples:
```bash
$ docker network create my_network
85ca267e928a826a0c2c6ef7136a96b1f50cd29e80c2bdb6e360b5fc73a7ee97
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
49481817aed2        bridge              bridge              local
ef3e0336719d        host                host                local
85ca267e928a        my_network          bridge              local
3ce4f11ad443        none                null                local
$ docker container run -d --name new_nginx --network my_network -p 8080:80 nginx:alpine
```

Attaching container to a network:
```bash
$ docker container run -d --name nginx3 -p 8080:80 nginx:alpine
104c8f401085e0fc5ba5eba18b24ec0540fed1959d1267f3d296f23ba82df9ef
$ docker network connect my_network 104c8
$
```

### Docker networks: DNS

Docker use container names as a DNS names to allow containers to talk to each other over the virtual network (inside the network).
```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
49481817aed2        bridge              bridge              local
ef3e0336719d        host                host                local
85ca267e928a        my_network          bridge              local
3ce4f11ad443        none                null                local
$ docker container run -d --name nginx_1 -p 80:80 --network my_network nginx:alpine
592e197b03e4c6e061338b9c18d6669a846ca088f5087c099b3212648949671e
$ docker container run -d --name nginx_2 -p 8080:80 --network my_network nginx:alpine
c444d5dd6bb2093f6dab6b5dcba3043460124c4a34bc49c02f9308ae5a80f076
$ docker container exec -it nginx_2 ping nginx_1
PING nginx_1 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.683 ms
64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.118 ms
64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.230 ms
^C
--- nginx_1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.118/0.343/0.683 ms
$
```

**Note 1:** a command to **ping another container**: `docker container exec -it nginx_2 ping nginx_1`

**Note 2:** The default network (bridge) **does not** have a DNS server built in. In this case the command `docker container create --list` could be used.

#### DNS round-robin

We can have multiple containers on a created network respond to the same DNS address.

Use `--network-alias` flag.

```bash
$ docker network create dude
b34ccd362696cfbd8d73cc78ff8568c21efade93ddd86f829974a67f07209e63
$ docker container run -d --net dude --net-alias search elasticsearch:2
f2ea89e02ddc06c75e8f504ca7b58fc6aa4dd391fa269a2b8269ff0cefc9f43b
$ docker container run -d --net dude --net-alias search elasticsearch:2
025bc98201c28228068e37ad4ed529157f90a8f955bba884276d4bf68c754919
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
025bc98201c2        elasticsearch:2     "/docker-entrypoint.…"   23 seconds ago      Up 21 seconds       9200/tcp, 9300/tcp   hardcore_nash
f2ea89e02ddc        elasticsearch:2     "/docker-entrypoint.…"   57 seconds ago      Up 56 seconds       9200/tcp, 9300/tcp   optimistic_borg
$ docker container run --rm --net dude alpine nslookup search
nslookup: can't resolve '(null)': Name does not resolve

Name:      search
Address 1: 172.19.0.3 search.dude
Address 2: 172.19.0.2 search.dude
$ docker container run --rm --net dude centos curl -s search:9200
{
  "name" : "Doctor Leery",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "AK4M9T-mQrK8yuTo9heapA",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
$ docker container run --rm --net dude centos curl -s search:9200
{
  "name" : "Sybil Dorn",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "mlPtiHDBQQytsbr-vD1jMQ",
  "version" : {
    "number" : "2.4.6",
    "build_hash" : "5376dca9f70f3abef96a77f4bb22720ace8240fd",
    "build_timestamp" : "2017-07-18T12:17:44Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.4"
  },
  "tagline" : "You Know, for Search"
}
```

## Docker images

Docker image, the **unofficial** defenition:
**Docker image** IS the app binaries and dependencies AND metadata about the image data and how to run the image.

Image is not a complete OS. It doesn't have kernel, kernel modules (drivers) etc... Host provides a kernel.


* list local images: `docker image ls`
* pull image: `docker image pull <image-name>:<image-version>` or just `docker image pull <image-name>`

`docker image history <image-name>[:<image-version>]` - shows layers of changes made in image

`docker image inspect <image>` - detailed info about the image's metadata in JSON format

### Docker hub, tagging and pushing

```bash
$ docker pull mysql/mysql-server
```

`$ docker image tag` - has a lot in common with `git tag`

`docker image tag <source-image>[:<tag>] <target-image>[:<tag>]` example:

The default tag (if <tag> was skipped) is **latest**, although it's just a last layer in the image, like a HEAD in git.


`docker login` - logs in to docker hub and stores the authentication key in user's profile as a file.

`docker logout` - logs out and deletes file with auth key. USE IT in prod environment or on the shared/multi-user system.

`docker push <image>` - it uploads and creates a new repo on docker hub if such repo didn't exist.

Example:

```bash
$ docker push dyakonoff/nginx
```


Creating a new image/repo & tag=latest from the standard repo:

```bash
$ docker image tag nginx dyakonoff/my_nginx
```

Tagging existing image (re-tagging):

```bash
$ docker image tag dyakonoff/nginx dyakonoff/nginx:testing
```

## Docker file

`Dockerfile` - is just a convention/default name. If needed, any file name can be used with `-f` flag: `docker build -f some-dockerfile`

`docker image build -t customnginx .` - build z Dockerfile locally.

## Docker Volumes

```bash
$ docker volume ls
DRIVER              VOLUME NAME
local               bbde38153b472fcb3c048925f0ea66d0776c80a9dc1dd5b5807f11c97e9ff962
local               d24c52ae324abc6984aa01b6ce92268af7dcf69de4bbfc8841048cfc231fb679
```

Named volumes, they also persist data between containers:

```bash
$ docker container run -d --name mysql -e MYSQL_ALLOW_EMPTY_PASSWORD=True -v mysql-db:/var/lib/mysql mysql
2612d31c557f0a89ae2001672040c0a5ed95d2231fcec9106345989a28719486
03:24:12 dyakonov@MacBook-Pro-dyakonov:~/projects/mydocs/docker-cheatsheet (master)*$ docker volume ls
DRIVER              VOLUME NAME
local               bbde38153b472fcb3c048925f0ea66d0776c80a9dc1dd5b5807f11c97e9ff962
local               d24c52ae324abc6984aa01b6ce92268af7dcf69de4bbfc8841048cfc231fb679
local               mysql-db
$
$ docker volume inspect mysql-db
[
    {
        "CreatedAt": "2019-04-23T23:24:28Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/mysql-db/_data",
        "Name": "mysql-db",
        "Options": null,
        "Scope": "local"
    }
]
```

`docker volume create` - is required to use different drivers and put labels on volumes.

## Bind Mounting

* It's a mapping of a host file or directory to a container file or directory.
* It skips UFS and host files overlaps any in container (if there are files wit the same path).
* Must be at `container run` command

Format of mapping: `docker container run ... -v <host-path>:<container-path>`

Bind mounts **starts with a forward slash or tilda?**.

Example:
```bash
$ docker container run -d --name mysql2 -e MYSQL_ALLOW_EMPTY_PASSWORD=True -v ~/projects/data/mysql-db:/var/lib/mysql mysql
a79f56e6d943bc84beca50b67c77651ee58fbafba38e6340f3008b93be031302
$ ls ~/projects/data/mysql-db/
#innodb_temp       binlog.000002      ca.pem             ib_buffer_pool     ibdata1            mysql.ibd          public_key.pem     sys
auto.cnf           binlog.index       client-cert.pem    ib_logfile0        ibtmp1             performance_schema server-cert.pem    undo_001
binlog.000001      ca-key.pem         client-key.pem     ib_logfile1        mysql              private_key.pem    server-key.pem     undo_002
```









## External links

### Containerization

* [eBook: Docker for the Virtualization Admins](https://github.com/mikegcoleman/docker101/blob/master/Docker_eBook_Jan_2017.pdf)
* [video: Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8&feature=youtu.be&list=PLBmVKD7o3L8v7Kl_XXh3KaJl9Qw2lyuFl)

### Commands for Getting Into The Local Docker VM
* [Docker for Mac Commands](https://www.bretfisher.com/docker-for-mac-commands-for-getting-into-local-docker-vm/)
* [Getting a Shell in the Docker for Windows Moby VM](https://www.bretfisher.com/getting-a-shell-in-the-docker-for-windows-vm/)


* [Package Management Basics: a pt, yum, dnf, pkg](https://www.digitalocean.com/community/tutorials/package-management-basics-apt-yum-dnf-pkg)

* [Format command and log output](https://docs.docker.com/config/formatting/)

### DNS basics

* [Funny comics about DNS basics](https://howdns.works/)
* [DNS: Why It’s Important and How It Works](https://dyn.com/blog/dns-why-its-important-how-it-works/)
* [Round-robin DNS](https://en.wikipedia.org/wiki/Round-robin_DNS)

### Docker images

* [Docker Image Specification v1.0.0](https://github.com/moby/moby/blob/master/image/spec/v1.md)
* [List of official docker images](https://github.com/docker-library/official-images/tree/master/library)
* [About storage drivers](https://docs.docker.com/storage/storagedriver/)
  
### Dockerfile reference

* [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)

### Security

* [Docker security cheat sheet (Russian)](https://habr.com/ru/company/acribia/blog/448704/)