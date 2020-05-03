# ElasticSearch-with-Docker-
Running a 3 Node Elasticsearch Cluster With Docker Compose
Pre-Requisites
We need to set the vm.max_map_count kernel parameter:
$ sudo sysctl -w vm.max_map_count=262144
To set this permanently, add it to /etc/sysctl.conf and reload with sudo sysctl -p
Docker Compose:
The docker compose file that we will reference:
Docker-compose.yml
version: '2.2'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.2.4
    container_name: elasticsearch
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata1:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - esnet
  elasticsearch2:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.2.4
    container_name: elasticsearch2
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata2:/usr/share/elasticsearch/data
    networks:
      - esnet
  elasticsearch3:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.2.4
    container_name: elasticsearch3
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - http.cors.enabled=true
      - http.cors.allow-origin=*
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.zen.ping.unicast.hosts=elasticsearch"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata3:/usr/share/elasticsearch/data
    networks:
      - esnet

  kibana:
    image: 'docker.elastic.co/kibana/kibana:6.3.2'
    container_name: kibana
    environment:
      SERVER_NAME: kibana.local
      ELASTICSEARCH_URL: http://elasticsearch:9200
    ports:
      - '5601:5601'
    networks:
      - esnet

  headPlugin:
    image: 'mobz/elasticsearch-head:5'
    container_name: head
    ports:
      - '9100:9100'
    networks:
      - esnet

volumes:
  esdata1:
    driver: local
  esdata2:
    driver: local
  esdata3:
    driver: local

networks:
  esnet:
The data of our elasticsearch container volumes will reside under /var/lib/docker, if you want them to persist in another location, you can use the driver_opts setting for the local volume driver.
Deploy
Deploy your elasticsearch cluster with docker compose:
$ docker-compose up
This will run in the foreground, and you should see console output.
Testing Elasticsearch
Let’s run a couple of queries, first up, check the cluster health api:

$ curl http://127.0.0.1:9200/_cluster/health?pretty
{
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 1,
  "active_shards" : 2,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}

Create a index with replication count of 2:
$ curl -H "Content-Type: application/json" -XPUT http://127.0.0.1:9200/test -d '{"number_of_replicas": 2}'
Ingest a document to elasticsearch:

$ curl -H "Content-Type: application/json" -XPUT http://127.0.0.1:9200/test/docs/1 -d '{"name": "ruan"}'
{"_index":"test","_type":"docs","_id":"1","_version":1,"result":"created","_shards":{"total":3,"successful":3,"failed":0},"_seq_no":0,"_primary_term":1}
View the indices:
$ curl http://127.0.0.1:9200/_cat/indices?v

health status index                       uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test                        w4p2Q3fTR4uMSYBfpNVPqw   5   2          1            0      3.3kb          1.1kb
green  open   .monitoring-es-6-2018.04.29 W69lql-rSbORVfHZrj4vug   1   1       1601 

Kibana
Kibana is also included in the stack and is accessible via http://localhost:5601/ and you it should look more or less like:

Elasticsearch Head UI
I always prefer working directly with the RESTFul API, but if you would like to use a UI to interact with Elasticsearch, you can access it via http://localhost:9100/ and should look like this:

Deleting the Cluster:
As its running in the foreground, you can just hit ctrl + c and as we persisted data in our compose, when you spin up the cluster again, the data will still be there.

