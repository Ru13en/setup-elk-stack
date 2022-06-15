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

Setup docker must be updated with DNS resolution:

`docker-compose build`

## Regenerate certificates
Use the following instructions base instructions:

https://github.com/deviantony/docker-elk/blob/tls/tls/README.md

At `instances.yml`, you must add to DNS records:

es.test2.thehip.app
kibana.test2.thehip.app

and to ip records

10.5.0.5
85.217.160.59

To generate the certificates, you must set the following steps:

```none
Generate a CSR? [y/N] n
Use an existing CA? [y/N] y
CA Path: /usr/share/elasticsearch/tls/ca/ca.p12
Password for ca.p12: <none>
For how long should your certificate be valid? [5y] 10y
Generate a certificate per node? [y/N] n
(Enter all the hostnames that you need, one per line.)
es.test2.thehip.app
kibana.test2.thehip.app
elasticsearch
localhost
Is this correct [Y/n] y
(Enter all the IP addresses that you need, one per line.)
10.5.0.5
10.5.0.6
10.5.0.7
127.0.0.1
85.217.160.59
Is this correct [Y/n] y
Do you wish to change any of these options? [y/N] n
Provide a password for the "http.p12" file: <none>
What filename should be used for the output zip file? tls/elasticsearch-ssl-http.zip
```

Unzip both files using sudo otherwise elastic search will delete your certificates due permission issues.

## Setup to start stack on boot and comunication between containers:
On docker-compose.yml you must add to services (elasticsearch, logstash and kibana):
`restart: always`

Add volume to logstash to receive config files `./logstash/conf.d/:/etc/logstash/conf.d/`.

I've had some issues regarding the DNS of docker and TLS certs authentication in the elasticsearch. 

To solve it could set an static ip address for each container in the stack:

```
version: '3.7'

services:

  # The 'setup' service runs a one-off script which initializes the
  # 'logstash_internal' and 'kibana_system' users inside Elasticsearch with the
  # values of the passwords defined in the '.env' file.
  #
  # This task is only performed during the *initial* startup of the stack. On all
  # subsequent runs, the service simply returns immediately, without performing
  # any modification to existing users.
  setup:
    build:
      context: setup/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    init: true
    volumes:
      - setup:/state:Z
    environment:
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:-}
      LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
      KIBANA_SYSTEM_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:-}
    networks:
      - elk

  elasticsearch:
    build:
      context: elasticsearch/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro,z
      - elasticsearch:/usr/share/elasticsearch/data:z
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: -Xms512m -Xmx512m
      # Bootstrap password.
      # Used to initialize the keystore during the initial startup of
      # Elasticsearch. Ignored on subsequent runs.
      ELASTIC_PASSWORD: ${ELASTIC_PASSWORD:-}
      # Use single node discovery in order to disable production mode and avoid bootstrap checks.
      # see: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
      discovery.type: single-node
    restart: always
    networks:
      elk:
        ipv4_address: 10.5.0.5

  logstash:
    build:
      context: logstash/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml:ro,Z
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
      -  ./logstash/conf.d/:/etc/logstash/conf.d/:ro,Z
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: -Xms256m -Xmx256m
      LOGSTASH_INTERNAL_PASSWORD: ${LOGSTASH_INTERNAL_PASSWORD:-}
    restart: always
    networks:
      elk:
        ipv4_address: 10.5.0.6
    depends_on:
      - elasticsearch

  kibana:
    build:
      context: kibana/
      args:
        ELASTIC_VERSION: ${ELASTIC_VERSION}
    volumes:
      - ./kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml:ro,Z
    ports:
      - "5601:5601"
    environment:
      KIBANA_SYSTEM_PASSWORD: ${KIBANA_SYSTEM_PASSWORD:-}
    restart: always
    networks:
      elk:
        ipv4_address: 10.5.0.7
    depends_on:
      - elasticsearch

networks:
  elk:
    ipam:
      driver: default
      config:
        - subnet: "10.5.0.0/24"

volumes:
  setup:
  elasticsearch:
```

## Configure logstash yml

The Kibana default configuration is stored in `logstash/config/logstash.yml`

Add the following lines:

monitoring.elasticsearch.hosts: "https://es.test2.thehip.app:9200"

## Configure kibana yml

The Kibana default configuration is stored in `kibana/config/kibana.yml`

Add the following lines:

server.publicBaseUrl: "https://kibana.test2.thehip.app"
elasticsearch.hosts: [ "https://es.test2.thehip.app:9200" ]

## Change default passwords

Configure passwords present in .env file using generated passwords (see. https://github.com/deviantony/docker-elk#initial-setup)

## Add logstash config file

In this example we only want to read the dpkg.log file and add it to elastic search.
Add dpkg.conf to `~/localhost/logstash/conf.d/`

```
input {
  file {
         path => "/var/log/dpkg.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
  filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}
}
output {
  elasticsearch {
    hosts => ["https://es.test2.thehip.app:9200"]
  }
}

```

Now you can can see in Kibana, go to Management → Kibana Index Patterns. Create a index pattern for `logstash-*` and you shoud be able to see all messages

## Statring elk-stack:
To start docker stack on boot and detachted you must -d option.

`sudo docker-compose up -d`

