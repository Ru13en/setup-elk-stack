# setup-elk-stack

## setup requirements (docker)

The machine doesn't have the docker engine, so must install using:

https://docs.docker.com/desktop/linux/install/ubuntu/

If docker-compose is not available by installing docker-compose-plugin, use:

`sudo apt install -y docker-compose`

Do not forget to add the user to docker group and make logout and login.

Add unzip package:

`sudo apt install -y unzip`

## Getting elk-stack with tls enabled:

`git clone --branch tls https://github.com/deviantony/docker-elk.git`

Configure passwords present in .env file:

Used password: nS?9N@4-&cCJ$2D?

## Regenerate certificates
Use the following instructions:

https://github.com/deviantony/docker-elk/blob/tls/tls/README.md

## Setup to start stack on boot:
On docker-compose.yml you must add to services (elasticsearch, logstash and kibana):
`restart: always`

## Configure kibana yml

The Kibana default configuration is stored in kibana/config/kibana.yml

Add the following lines:
elasticsearch.hosts: [ "https://elasticsearch:9200, https://es.test2.thehip.app:9200" ]
server.publicBaseUrl: "kibana.test2.thehip.app"


## Statring elk-stack:
To start docker stack on boot and detachted you must provide --restart always and -d options.

`docker-compose up -d`

