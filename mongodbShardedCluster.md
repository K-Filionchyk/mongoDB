Создаём VM в Google Cloud Platform с помощью gcloud

``` 
gcloud compute instances create mongo --project=semiotic-joy-349205 --zone=us-west4-b --machine-type=e2-medium --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=91004319699-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/devstorage.read_only,https://www.googleapis.com/auth/logging.write,https://www.googleapis.com/auth/monitoring.write,https://www.googleapis.com/auth/servicecontrol,https://www.googleapis.com/auth/service.management.readonly,https://www.googleapis.com/auth/trace.append --create-disk=auto-delete=yes,boot=yes,device-name=mongo,image=projects/ubuntu-os-cloud/global/images/ubuntu-2004-focal-v20220419,mode=rw,size=10,type=projects/semiotic-joy-349205/zones/us-west4-b/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any
```

Заходим в VM

``` gcloud compute ssh mongo ```

Устанавливаем mongo

``` 
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add - && echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list && sudo apt-get update && sudo apt-get install -y mongodb-org
```

Создадим репликасет с конфигурацией шарда
```
sudo mkdir /home/mongo && sudo mkdir /home/mongo/{dbc1,dbc2,dbc3} && sudo chmod 777 /home/mongo/{dbc1,dbc2,dbc3}
mongod --configsvr --dbpath /home/mongo/dbc1 --port 27001 --replSet RScfg --fork --logpath /home/mongo/dbc1/dbc1.log --pidfilepath /home/mongo/dbc1/dbc1.pid
mongod --configsvr --dbpath /home/mongo/dbc2 --port 27002 --replSet RScfg --fork --logpath /home/mongo/dbc2/dbc2.log --pidfilepath /home/mongo/dbc2/dbc2.pid
mongod --configsvr --dbpath /home/mongo/dbc3 --port 27003 --replSet RScfg --fork --logpath /home/mongo/dbc3/dbc3.log --pidfilepath /home/mongo/dbc3/dbc3.pid
```

инициируем репликасет
```
rs.initiate({"_id" : "RScfg", configsvr: true, members : [{"_id" : 0, priority : 3, host : "127.0.0.1:27001"},{"_id" : 1, host : "127.0.0.1:27002"},{"_id" : 2, host : "127.0.0.1:27003"}]});
```


Создаём ещё 3 репликасета

```
sudo mkdir /home/mongo/{db1,db2,db3,db4,db5,db6,db7,db8,db9} && sudo chmod 777 /home/mongo/{db1,db2,db3,db4,db5,db6,db7,db8,db9}
mongod --shardsvr --dbpath /home/mongo/db1 --port 27011 --replSet RS1 --fork --logpath /home/mongo/db1/db1.log --pidfilepath /home/mongo/db1/db1.pid
mongod --shardsvr --dbpath /home/mongo/db2 --port 27012 --replSet RS1 --fork --logpath /home/mongo/db2/db2.log --pidfilepath /home/mongo/db2/db2.pid
mongod --shardsvr --dbpath /home/mongo/db3 --port 27013 --replSet RS1 --fork --logpath /home/mongo/db3/db3.log --pidfilepath /home/mongo/db3/db3.pid
mongod --shardsvr --dbpath /home/mongo/db4 --port 27021 --replSet RS2 --fork --logpath /home/mongo/db4/db4.log --pidfilepath /home/mongo/db4/db4.pid
mongod --shardsvr --dbpath /home/mongo/db5 --port 27022 --replSet RS2 --fork --logpath /home/mongo/db5/db5.log --pidfilepath /home/mongo/db5/db5.pid
mongod --shardsvr --dbpath /home/mongo/db6 --port 27023 --replSet RS2 --fork --logpath /home/mongo/db6/db6.log --pidfilepath /home/mongo/db6/db6.pid
mongod --shardsvr --dbpath /home/mongo/db7 --port 27031 --replSet RS3 --fork --logpath /home/mongo/db7/db7.log --pidfilepath /home/mongo/db7/db7.pid
mongod --shardsvr --dbpath /home/mongo/db8 --port 27032 --replSet RS3 --fork --logpath /home/mongo/db8/db8.log --pidfilepath /home/mongo/db8/db8.pid
mongod --shardsvr --dbpath /home/mongo/db9 --port 27033 --replSet RS3 --fork --logpath /home/mongo/db9/db9.log --pidfilepath /home/mongo/db9/db9.pid
```

```
mongo --port 27011
rs.initiate({"_id" : "RS1", members : [{"_id" : 0, priority : 3, host : "127.0.0.1:27011"},{"_id" : 1, host : "127.0.0.1:27012"},{"_id" : 2, host : "127.0.0.1:27013", arbiterOnly : true}]});

mongo --port 27021
rs.initiate({"_id" : "RS2", members : [{"_id" : 0, priority : 3, host : "127.0.0.1:27021"},{"_id" : 1, host : "127.0.0.1:27022"},{"_id" : 2, host : "127.0.0.1:27023", arbiterOnly : true}]});

mongo --port 27031
rs.initiate({"_id" : "RS3", members : [{"_id" : 0, priority : 3, host : "127.0.0.1:27031"},{"_id" : 1, host : "127.0.0.1:27032"},{"_id" : 2, host : "127.0.0.1:27033", arbiterOnly : true}]});
```


Создадим шардированный кластер (запускаем в 2 экземплярах для отказоустойчивости)
```
mongos --configdb RScfg/127.0.0.1:27001,127.0.0.1:27002,127.0.0.1:27003 --port 27000 --fork --logpath /home/mongo/dbc1/dbs.log --pidfilepath /home/mongo/dbc1/dbs.pid
mongos --configdb RScfg/127.0.0.1:27001,127.0.0.1:27002,127.0.0.1:27003 --port 27100 --fork --logpath /home/mongo/dbc1/dbs2.log --pidfilepath /home/mongo/dbc1/dbs2.pid 
```

добавляем 3 шарда
```
mongo --port 27000
sh.addShard("RS1/127.0.0.1:27011,127.0.0.1:27012,127.0.0.1:27013")
sh.addShard("RS2/127.0.0.1:27021,127.0.0.1:27022,127.0.0.1:27023")
sh.addShard("RS3/127.0.0.1:27031,127.0.0.1:27032,127.0.0.1:27033")
```

проверяем
```
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("62bc6b962742e7ea6a38035d")
  }
  shards:
        {  "_id" : "RS1",  "host" : "RS1/127.0.0.1:27011,127.0.0.1:27012",  "state" : 1 }
        {  "_id" : "RS2",  "host" : "RS2/127.0.0.1:27021,127.0.0.1:27022",  "state" : 1 }
        {  "_id" : "RS3",  "host" : "RS3/127.0.0.1:27031,127.0.0.1:27032",  "state" : 1 }
  active mongoses:
        "4.4.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
  databases:
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
mongos> 
```

генерируем данные
```
use bank
sh.enableSharding("bank")
use config
db.settings.save({ _id:"chunksize", value: 1})
use bank
for (var i=0; i<100000; i++) { db.tickets.insert({name: "Max ammout of cost tickets", amount: Math.random()*100}) }
```

создаём индекс
```db.tickets.createIndex({amount: 1})```

включаем шардирование
```sh.shardCollection("bank.tickets",{amount: 1})```

проверяем
```
mongos> sh.status()
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("62bc6b962742e7ea6a38035d")
  }
  shards:
        {  "_id" : "RS1",  "host" : "RS1/127.0.0.1:27011,127.0.0.1:27012",  "state" : 1 }
        {  "_id" : "RS2",  "host" : "RS2/127.0.0.1:27021,127.0.0.1:27022",  "state" : 1 }
        {  "_id" : "RS3",  "host" : "RS3/127.0.0.1:27031,127.0.0.1:27032",  "state" : 1 }
  active mongoses:
        "4.4.13" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                329 : Success
  databases:
        {  "_id" : "bank",  "primary" : "RS2",  "partitioned" : true,  "version" : {  "uuid" : UUID("97c302b4-b162-4e14-9f3e-2aa12705da7d"),  "lastMod" : 1 } }
                bank.tickets
                        shard key: { "amount" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                RS1	2
                                RS2	3
                                RS3	2
                        { "amount" : { "$minKey" : 1 } } -->> { "amount" : 14.963406741297636 } on : RS3 Timestamp(2, 0) 
                        { "amount" : 14.963406741297636 } -->> { "amount" : 29.890858753854854 } on : RS1 Timestamp(3, 0) 
                        { "amount" : 29.890858753854854 } -->> { "amount" : 45.13565991300832 } on : RS3 Timestamp(4, 0) 
                        { "amount" : 45.13565991300832 } -->> { "amount" : 58.77100393559586 } on : RS1 Timestamp(5, 0) 
                        { "amount" : 58.77100393559586 } -->> { "amount" : 72.54771432540188 } on : RS2 Timestamp(5, 1) 
                        { "amount" : 72.54771432540188 } -->> { "amount" : 86.27035538697754 } on : RS2 Timestamp(1, 5) 
                        { "amount" : 86.27035538697754 } -->> { "amount" : { "$maxKey" : 1 } } on : RS2 Timestamp(1, 6) 
        {  "_id" : "config",  "primary" : "config",  "partitioned" : true }
                config.system.sessions
                        shard key: { "_id" : 1 }
                        unique: false
                        balancing: true
                        chunks:
                                RS1	699
                                RS2	163
                                RS3	162
                        too many chunks to print, use verbose if you want to force print
        {  "_id" : "test",  "primary" : "RS3",  "partitioned" : false,  "version" : {  "uuid" : UUID("2fe92f47-4993-4cf8-9b1e-4e83897df425"),  "lastMod" : 1 } }
mongos> 

```

Создаём роль и пользователя, запускаем сервер с аутентификацией
```
db = db.getSiblingDB("bank")
mongos> db.createRole(
...     {      
...         role: "BankEmployee",      
...         privileges:[{ resource: {db:"bank", collection: ""}, actions: ["anyAction"]}],      
...         roles:[{ role: "readWrite", db: "bank" }]
...     }
... )
{
	"role" : "BankEmployee",
	"privileges" : [
		{
			"resource" : {
				"db" : "bank",
				"collection" : ""
			},
			"actions" : [
				"anyAction"
			]
		}
	],
	"roles" : [
		{
			"role" : "readWrite",
			"db" : "bank"
		}
	]
}
mongos> 
```


```
db.createUser({      
    user: "bankAdmin",      
    pwd: "qwerty",      
    roles: ["BankEmployee"]
})
```

mongo --port 27000 -u bankAdmin -p qwerty --authenticationDatabase "bank"

```
filenchik@mongo:~$ mongo --port 27000 -u bankAdmin -p qwerty --authenticationDatabase "bank"
MongoDB shell version v4.4.13
connecting to: mongodb://127.0.0.1:27000/?authSource=bank&compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("f190dfd9-02ee-4f3e-98e1-018a47f6bc9c") }
MongoDB server version: 4.4.13
---
The server generated these startup warnings when booting: 
        2022-06-29T15:39:01.344+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
        2022-06-29T15:39:01.344+00:00: This server is bound to localhost. Remote systems will be unable to connect to this server. Start the server with --bind_ip <address> to specify which IP addresses it should serve responses from, or with --bind_ip_all to bind to all interfaces. If this behavior is desired, start the server with --bind_ip 127.0.0.1 to disable this warning
---
mongos> use bank
switched to db bank
mongos> db.tickets.find()
{ "_id" : ObjectId("62bc74955360a69187e7d010"), "name" : "Max ammout of cost tickets", "amount" : 64.39249572696528 }
{ "_id" : ObjectId("62bc74955360a69187e7d011"), "name" : "Max ammout of cost tickets", "amount" : 66.21174353473978 }
{ "_id" : ObjectId("62bc74955360a69187e7d013"), "name" : "Max ammout of cost tickets", "amount" : 95.08585458499022 }
{ "_id" : ObjectId("62bc74955360a69187e7d017"), "name" : "Max ammout of cost tickets", "amount" : 66.6819565243121 }
{ "_id" : ObjectId("62bc74955360a69187e7d01c"), "name" : "Max ammout of cost tickets", "amount" : 71.6842544450924 }
{ "_id" : ObjectId("62bc74955360a69187e7d020"), "name" : "Max ammout of cost tickets", "amount" : 68.06733562905194 }
{ "_id" : ObjectId("62bc74955360a69187e7d025"), "name" : "Max ammout of cost tickets", "amount" : 91.17731050107984 }
{ "_id" : ObjectId("62bc74955360a69187e7d027"), "name" : "Max ammout of cost tickets", "amount" : 60.97582567250378 }
{ "_id" : ObjectId("62bc74955360a69187e7d028"), "name" : "Max ammout of cost tickets", "amount" : 85.4628103474074 }
{ "_id" : ObjectId("62bc74955360a69187e7d02a"), "name" : "Max ammout of cost tickets", "amount" : 70.28485106589496 }
{ "_id" : ObjectId("62bc74955360a69187e7d02f"), "name" : "Max ammout of cost tickets", "amount" : 96.38769600773018 }
{ "_id" : ObjectId("62bc74955360a69187e7d033"), "name" : "Max ammout of cost tickets", "amount" : 77.03519469338204 }
{ "_id" : ObjectId("62bc74955360a69187e7d036"), "name" : "Max ammout of cost tickets", "amount" : 78.8176472125406 }
{ "_id" : ObjectId("62bc74955360a69187e7d03f"), "name" : "Max ammout of cost tickets", "amount" : 75.87876692907084 }
{ "_id" : ObjectId("62bc74955360a69187e7d047"), "name" : "Max ammout of cost tickets", "amount" : 97.6307637323546 }
{ "_id" : ObjectId("62bc74955360a69187e7d04d"), "name" : "Max ammout of cost tickets", "amount" : 58.865786970886234 }
{ "_id" : ObjectId("62bc74955360a69187e7d04f"), "name" : "Max ammout of cost tickets", "amount" : 69.21900452126108 }
{ "_id" : ObjectId("62bc74955360a69187e7d051"), "name" : "Max ammout of cost tickets", "amount" : 65.36821476797738 }
{ "_id" : ObjectId("62bc74955360a69187e7d053"), "name" : "Max ammout of cost tickets", "amount" : 75.65737571776087 }
{ "_id" : ObjectId("62bc74955360a69187e7d054"), "name" : "Max ammout of cost tickets", "amount" : 89.32753999772045 }
Type "it" for more
mongos>
```