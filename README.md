# Description

PoC About create a MDM Elastic 7.4.2 to Elastic 7.16.3 using Snapshots. We will detail all steps to execute this process

## STEP01: Start mdm with elastic 7.4.2

Please download the repo: https://github.com/masalinas/poc-mdm-haproxy-oauth2 to start the MDM services and the Elastic 7.4.2 with MDM indices
to be exported

```
$ docker compose up -d --no-deps --build
```

## STEP02: List elastic indices

We can list the original MDM indices to be exported in the MDM Elastic service executing this REST API Endpoint

```
GET http://localhost:9200/_cat/indices?v=true&s=index
```

## STEP03: Create a respository for your snapshots

Execute this REST API endpoint to create a repository for your snapshots

```
PUT http://localhost:9200/_snapshot/my-mdm_backup
{
  "type": "fs",
  "settings": {
    "location": "mdm_backup"
  }
}
```

## STEP4: Create a snapshot

We create manually a snaphost of my elastic instance

```
PUT http://localhost:9200/_snapshot/my-mdm_backup/my_snapshot_20240719154610
```

## STEP05: Start new elasticsearch

Now we are gping to start another new ElasticSearch service to import the previous snaphots.
We my must:
- Select a unique name for this service
- Select a new port for this service: 9201
- Add ***path.repo** environment variable with a folder where recover the snapshots: /usr/share/elasticsearch/backup
- Add a volumen attached to the previous volume where the MDM save its snapshots: poc-haproxy-oauth2_elastic_backup

```
$ docker run -d \
--name consum-elasticsearch-new \
-p 9201:9200 -p 9300:9300 \
-e "discovery.type=single-node" \
-e "path.repo=/usr/share/elasticsearch/backup" \
--volume poc-haproxy-oauth2_elastic_backup:/usr/share/elasticsearch/backup \
--net consum \
elasticsearch:7.16.3
```

## STEP06: List elastic indices

List the indices of the new ElasticSearch to check the original indices previous the import step

```
GET http://localhost:9201/_cat/indices?v=true&s=index
```

## STEP07: Create a respository for your snapshots

Execute this REST API endpoint to create a repository for your snapshots. We can select the same name or diferent fot this repository. 
The must important is select the same location used previous in the MDM service where the snapshots are saved

```
PUT http://localhost:9201/_snapshot/my-mdm_backup
{
  "type": "fs",
  "settings": {
    "location": "mdm_backup"
  }
}
```

## STEP08:  List the snapshot repositories

List the repositories created for this service

```
GET http://localhost:9201/_snapshot
```


## STEP09:  List the snapshots inside the repositories

List the snapshot repositories created for this service. We will see all snapshots created for the MDM service

```
GET http://localhost:9201/_snapshot/my-mdm_backup/*?verbose=false
```

## STEP10:  Restore a snapshot

We will select what snapshot to be import from the previous list

```
POST http://localhost:9201/_snapshot/my-mdm_backup/my_snapshot_20240719154610/_restore
```

## STEP11:  List the snapshot repositories

List again the indices and we will see the new indices from MDM in out new elastic search service correctly:

```
GET http://localhost:9201/_cat/indices?v=true&s=index
```