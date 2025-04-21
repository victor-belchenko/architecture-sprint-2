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

docker exec -it shard1-1 mongosh --port 27018

> rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id: 0, host: "173.17.0.9:27018" }, 
        { _id: 1, host: "173.17.0.29:27018" },  
        { _id: 2, host: "173.17.0.39:27018" }
      ]
    }
);
> exit();


docker exec -it shard2-1 mongosh --port 27019

> rs.initiate(
    {
      _id : "shard2",
      members: [
       { _id: 0, host: "173.17.0.8:27019" }, 
        { _id: 1, host: "173.17.0.28:27019" },  
        { _id: 2, host: "173.17.0.38:27019" }
      ]
    }
  );
> exit(); 

4. Инициализировать роутер и наполнить данными

docker exec -it mongos_router mongosh --port 27020

> sh.addShard("shard1/173.17.0.9:27018,173.17.0.29:27018,173.17.0.39:27018"); 

> sh.addShard("shard2/173.17.0.8:27019,173.17.0.28:27019,173.17.0.38:27019"); 

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
 docker exec -it shard1-1 mongosh --port 27018
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit(); 

 docker exec -it shard2-1 mongosh --port 27019
 > use somedb;
 > db.helloDoc.countDocuments();
 > exit();
7. Инициировать redis
docker exec -it redis redis-cli -h 173.17.0.2 -p 6379
8. Запуск приложения на компьютере
uvicorn api_app.app:app --reload
9. Очистить кэш
docker exec -it redis redis-cli FLUSHALL
проверить результат
docker exec -it redis redis-cli KEYS '*'
проверить время выполнения запроса
time curl -o /dev/null -s -w "%{time_total}\n" http://localhost:8080/helloDoc/users
