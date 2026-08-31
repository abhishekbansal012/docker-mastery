# Important Commands in DOCKER SWARM

## Initiate docker Swarm( By default swarm is disabled) 
docker swarm init

## Create a docker service 
docker service create alpine ping 8.8.8.8

## List docker services
docker service ls

## Details of services
docker service ps <ID>

docker container ls

docker service update n11j0u0qicdn --replicas 3