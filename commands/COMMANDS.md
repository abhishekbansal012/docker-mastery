# Important Commands in DOCKER

## View docker version
docker version

## View docker info
docker info

## View docker commands available (basic)
docker

## Create a new container
docker container run --publish 80:80 nginx

## Create a new container Detached Mode
docker container run --publish 80:80 --detach nginx

## Create a new container Detached Mode with a custom name
docker container run --publish 80:80 --detach --name webhost nginx

## List all containers RUNNING
docker container ls

## List all containers ALL
docker container ls -a

## Stop a container using its id
docker container stop <ID>

## View docker logs
docker container logs <ID>

## Remove docker containers 
### we can give multiple ids here 
docker container rm <ID> <ID> <ID>
eg - docker container rm bbdd2a86cd18 d501d7e10c96 67e7c01c7b9f

## View which port a container is using and exposing to physical network 
docker container port webhost

## view mac ipconfig details(Commpand is IF CONFIG not IP  CONFIG)
ifconfig en0

## list all network in docker
docker network ls

## Inspect a network
docker network inspect

## create a network
docker network create --driver

## Attach a network to container
docker network connect

## Detach a network from a container
docker network disconnect

## List Docker Images
docker image ls


