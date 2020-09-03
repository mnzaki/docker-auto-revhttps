- create the network and volume first:
  docker-network create nginx-proxy
  docker volume create --name=nginx-static
- Try to add smart helper scripts (e.g. wrap docker-machine env stuff in a function)
