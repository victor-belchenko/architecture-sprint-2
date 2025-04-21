Для инициализации MongoDB в шардированном варианте:
1. Был доработан compose.yaml:
- настройки сервисов configSrv, shard1, shard2, mongos_router - по примеру данному в уроке "Реализация шардирования"
- в настройку сервиса pymongo_api добавлено явное определение ip адреса ipv4_address: 173.17.0.11
2. Переапустить докер 
docker-compose down -v
docker-compose up -d
3. Инициализировать конфигурационный сервер и шарды 
docker exec -it configSrv mongosh --port 27017

> rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);
> exit();

docker exec -it shard1 mongosh --port 27018

> rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" },
       // { _id : 1, host : "shard2:27019" }
      ]
    }
);
> exit();

docker exec -it shard2 mongosh --port 27019

> rs.initiate(
    {
      _id : "shard2",
      members: [
       // { _id : 0, host : "shard1:27018" },
        { _id : 1, host : "shard2:27019" }
      ]
    }
  );
> exit(); 

4. Инициализировать роутер и наполнить данными

docker exec -it mongos_router mongosh --port 27020

> sh.addShard( "shard1/shard1:27018");
> sh.addShard( "shard2/shard2:27019");

> sh.enableSharding("somedb");
> sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )

> use somedb

> for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i})

> db.helloDoc.countDocuments() 
> exit(); 

5. Проверить отклик роутера в браузере
http://localhost:8080/
{"mongo_topology_type":"Single","mongo_replicaset_name":null,"mongo_db":"somedb","read_preference":"Primary()","mongo_nodes":[["mongodb1",27017]],"mongo_primary_host":null,"mongo_secondary_hosts":[],"mongo_address":["mongodb1",27017],"mongo_is_primary":true,"mongo_is_mongos":false,"collections":{"helloDoc":{"documents_count":1000}},"shards":null,"cache_enabled":false,"status":"OK"}
6. Проверить количество документов на шардах
 docker exec -it shard1 mongosh --port 27018
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit(); 

 docker exec -it shard2 mongosh --port 27019
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit();
