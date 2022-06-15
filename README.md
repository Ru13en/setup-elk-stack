# setup-elk-stack

## setup requirements
The machine doesn't have the docker engine, so must install using:
https://docs.docker.com/desktop/linux/install/ubuntu/

If docker-compose is not available by installing docker-compose-plugin, use:
`sudo apt install docker-compose`

Do not forget to add the user to docker group and make logout and login.

Getting elk-stack with tls enabled:
`git clone --branch tls https://github.com/deviantony/docker-elk.git`

## Statring elk-stack:
To start docker stack on boot and detachted you must provide --restart always and -d options.
`docker-compose up --restart always -d`

