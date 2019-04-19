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
```
docker container run -p 3306:3306 -d --name mysql -e MYSQL_RANDOM_ROOT_PASSWORD=true mysql
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
```
$ docker container start -ai <container-name/id>
```
Connect to a running container and run additional process (bash/sh in our case):
```
docker container exec -it mysql bash
docker container exec -it nginx sh
```




## External links

### Containerization
* [eBook: Docker for the Virtualization Admins](https://github.com/mikegcoleman/docker101/blob/master/Docker_eBook_Jan_2017.pdf)
* [video: Cgroups, namespaces, and beyond: what are containers made from?](https://www.youtube.com/watch?v=sK5i-N34im8&feature=youtu.be&list=PLBmVKD7o3L8v7Kl_XXh3KaJl9Qw2lyuFl)

## Commands for Getting Into The Local Docker VM
* [Docker for Mac Commands](https://www.bretfisher.com/docker-for-mac-commands-for-getting-into-local-docker-vm/)
* [Getting a Shell in the Docker for Windows Moby VM](https://www.bretfisher.com/getting-a-shell-in-the-docker-for-windows-vm/)


* [Package Management Basics: a pt, yum, dnf, pkg](https://www.digitalocean.com/community/tutorials/package-management-basics-apt-yum-dnf-pkg)


