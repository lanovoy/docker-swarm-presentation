# Docker Swarm Presentation

## Presentation
[Slides](http://www.slideshare.net/albertogviana/docker-swarm-71804647)


## Building a docker swarm cluster
```
docker-machine create -d virtualbox manager

for i in 1 2; do
    docker-machine create -d virtualbox worker-$i
done
```

Checking my machines
```
docker-machine ls
```

Creating the cluster
```
eval "$(docker-machine env manager)"

docker swarm init --advertise-addr $(docker-machine ip manager)
```

## Adding the visualizer service
```
eval "$(docker-machine env manager)"

docker service create \
  --name=visualizer \
  --publish=8000:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  --detach=false \
  dockersamples/visualizer

explorer http://$(docker-machine ip manager):8000
```

## Adding workers to the cluster
```
eval "$(docker-machine env manager)"
JOIN_TOKEN=$(docker swarm join-token -q worker)

for i in 1 2; do
    eval "$(docker-machine env worker-$i)"

    docker swarm join --token $JOIN_TOKEN \
        --advertise-addr $(docker-machine ip worker-$i) \
        $(docker-machine ip manager):2377
done
```

```
eval "$(docker-machine env manager)"
docker node ls
```

## Creating network
```
eval "$(docker-machine env manager)"
docker network create -d overlay routing-mesh
```

## Deploy a new service
```
eval "$(docker-machine env manager)"
docker service create \
  --name=docker-routing-mesh \
  --publish=8080:8080/tcp \
  --network routing-mesh \
  --reserve-memory 20m \
  lanovoy/docker-swarm-presentation:1.0
```

## Testing the service
```
curl http://$(docker-machine ip manager):8080
curl http://$(docker-machine ip worker-1):8080
curl http://$(docker-machine ip worker-2):8080
```

## Scaling a service
```
docker service scale docker-routing-mesh=3
```

## Calling a service
```
while true; do curl http://$(docker-machine ip manager):8080; sleep 1; printf "\n";  done
```

## Rolling updates
```
eval "$(docker-machine env manager)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --image lanovoy/docker-swarm-presentation:2.0 \
  docker-routing-mesh
```

## Calling a service
```
while true; do curl http://$(docker-machine ip manager):8080/health; sleep 1; printf "\n";  done
```

## Docker secret create
```
docker-machine.exe ssh manager
```

```
echo my-very-secret-value | docker secret create my_secret -
```

## Docker secret ls
```
docker secret ls
```

## Deploy my secret
```
eval "$(docker-machine env manager)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --secret-add my_secret \
  --image lanovoy/docker-swarm-presentation:2.0 \
  docker-routing-mesh
```

```
curl http://$(docker-machine ip manager):8080
```

```
docker exec $(docker ps --filter name=docker-routing-mesh -q) ls -l /run/secrets
```

## Drain a node
```
docker node update --availability=drain worker-2
```
```
while true; do curl http://$(docker-machine ip manager):8080/health; sleep 1; printf "\n";  done
```

## Listing nodes
```
docker node ls
```

## Bring the node back
```
docker node update --availability=active worker-2
```

