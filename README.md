# Docker Swarm Presentation

## Presentation
[Slides](http://www.slideshare.net/albertogviana/docker-SalesPortalSPA71804647)


## Building a docker swarm cluster
```
docker-machine create -d virtualbox manager-node

for i in 1 2; do
    docker-machine create -d virtualbox worker-node-$i
done
```

Checking my machines
```
docker-machine ls
```

Creating the cluster
```
eval "$(docker-machine env manager-node)"

docker swarm init --advertise-addr $(docker-machine ip manager-node)
```

## Adding the visualizer service
```
eval "$(docker-machine env manager-node)"

docker service create \
  --name=visualizer \
  --publish=8000:8080/tcp \
  --constraint=node.role==manager \
  --mount=type=bind,src=/var/run/docker.sock,dst=/var/run/docker.sock \
  dockersamples/visualizer

explorer http://$(docker-machine ip manager-node):8000
```

## Adding workers to the cluster
```
eval "$(docker-machine env manager-node)"
JOIN_TOKEN=$(docker swarm join-token -q worker)

for i in 1 2; do
    eval "$(docker-machine env worker-node-$i)"

    docker swarm join --token $JOIN_TOKEN \
        --advertise-addr $(docker-machine ip worker-node-$i) \
        $(docker-machine ip manager-node):2377
done
```

```
eval "$(docker-machine env manager-node)"
docker node ls
```

## Creating network
```
eval "$(docker-machine env manager-node)"
docker network create -d overlay routing-mesh
```

## Deploy a new service
```
eval "$(docker-machine env manager-node)"
docker service create \
  --name=docker-routing-mesh \
  --publish=8080:8080/tcp \
  --network routing-mesh \
  --reserve-memory 20m \
  lanovoy/docker-swarm-presentation:1.0
```

## Testing the service
```
curl http://$(docker-machine ip manager-node):8080
curl http://$(docker-machine ip worker-node-1):8080
curl http://$(docker-machine ip worker-node-2):8080
```

## Scaling a service
```
docker service scale docker-routing-mesh=3
```

## Calling a service
```
while true; do curl http://$(docker-machine ip manager-node):8080; sleep 1; printf "\n";  done
```

## Rolling updates
```
eval "$(docker-machine env manager-node)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --image lanovoy/docker-swarm-presentation:2.0 \
  docker-routing-mesh
```

## Calling a service
```
while true; do curl http://$(docker-machine ip manager-node):8080/health; sleep 1; printf "\n";  done
```

## Docker secret create
```
echo my-very-secret-value | docker secret create my_secret -
```

## Docker secret ls
```
docker secret ls
```

## Deploy my secret
```
eval "$(docker-machine env manager-node)"
docker service update \
  --update-failure-action pause \
  --update-parallelism 1 \
  --secret-add my_secret \
  --image lanovoy/docker-swarm-presentation:2.0 \
  docker-routing-mesh
```

```
curl http://$(docker-machine ip manager-node):8080
```

## Drain a node
```
docker node update --availability=drain worker-node-2
```

## Listing nodes
```
docker node ls
```

## Bring the node back
```
docker node update --availability=active worker-node-2
```

