# 另一种配置方法
> https://liking.site/2023/10/11/%e3%80%90%e6%9c%80%e4%bd%b3%e5%ae%9e%e8%b7%b5%e3%80%91%e9%ab%98%e5%8f%af%e7%94%a8mongodb%e9%9b%86%e7%be%a41%e5%88%86%e7%89%873%e5%89%af%e6%9c%ac%e8%a7%84%e5%88%92%e5%8f%8a%e9%83%a8%e7%bd%b2/

# 准备工作

## 设定项目根目录
```bash
MongoDir=/home/barry/workspaces/ops/mongo-cluster
```

## 目标文件结构
```bash
# 只有 400 999:999 mongo.keyfile, 其余 777 barry:barry 
(base) ➜  mongo-cluster tree ${MongoDir} -L 2 --dirsfirst
.
├── conf
│   ├── config.conf
│   ├── mongo.keyfile
│   ├── mongos.conf
│   ├── shard1.conf
│   ├── shard2.conf
│   └── shard3.conf
├── config
│   ├── 1
│   ├── 2
│   └── 3
├── mongos
│   ├── 1
│   └── 2
├── shard1
│   ├── 1
│   ├── 2
│   └── 3
├── shard2
│   ├── 1
│   ├── 2
│   └── 3
├── shard3
│   ├── 1
│   ├── 2
│   └── 3
├── docker-compose.repl.yml
├── docker-compose.sharding.yml
└── docker-compose.yml
```

## 准备文件夹
```bash
cd ${MongoDir}

mkdir -p conf

mkdir -p mongos/1/log
mkdir -p mongos/2/log

mkdir -p config/1/data
mkdir -p config/1/log
mkdir -p config/2/data
mkdir -p config/2/log
mkdir -p config/3/data
mkdir -p config/3/log

mkdir -p shard1/1/log
mkdir -p shard1/2/data
mkdir -p shard1/2/log
mkdir -p shard1/3/data
mkdir -p shard1/3/log

mkdir -p shard2/1/log
mkdir -p shard2/2/data
mkdir -p shard2/2/log
mkdir -p shard2/3/data
mkdir -p shard2/3/log

mkdir -p shard3/1/log
mkdir -p shard3/2/data
mkdir -p shard3/2/log
mkdir -p shard3/3/data
mkdir -p shard3/3/log

# for security
sudo chown 999:999 conf/mongo.keyfile
# for container gen pid
sudo chown 999:999 config/1 config/2 config/3 mongos/1 mongos/2 shard1/1 shard1/2 shard1/3 shard2/1 shard2/2 shard2/3 shard3/1 shard3/2 shard3/3
# for log
sudo chown 999:999 config/1/log config/2/log config/3/log mongos/1/log mongos/2/log shard1/1/log shard1/2/log shard1/3/log shard2/1/log shard2/2/log shard2/3/log shard3/1/log shard3/2/log shard3/3/log
```

## 安全认证-生成密钥
```bash
cd ${MongoDir}/conf
openssl rand -out mongo.keyfile -base64 90
chmod 400 mongo.keyfile
# for docker
chown 999:999 mongo.keyfile
```

## 启动容器
```bash
docker-compose -f docker-compose.sharding.yml up 
```

# 配置 MongoDB 容器

## 配置 Config 节点副本集
```bash
docker exec -it mongo_config1 mongosh 127.0.0.1:21000
```

```javascript
config = {_id: "configs", members: [
    {_id: 0, host: "config1:21000"},
    {_id: 1, host: "config2:21000"},
    {_id: 2, host: "config3:21000", arbiterOnly:true}
]}
rs.initiate(config)
rs.stats()
```

## 配置 Shard 节点副本集

### Shard1
```bash
docker exec -it mongo_shard1.1 mongosh 127.0.0.1:27001
```

```javascript
config = {_id: "shard1", members: [
    {_id: 0, host: "shard1.1:27001"},
    {_id: 1, host: "shard1.2:27001"},
    {_id: 2, host: "shard1.3:27001", arbiterOnly:true}
]}
rs.initiate(config)
rs.stats()
```

### Shard2
```bash
docker exec -it mongo_shard2.1 mongosh 127.0.0.1:27002
```

## configure shard2 nodes as a replicaSet
```javascript
config = {_id: "shard2", members: [
    {_id: 0, host: "shard2.1:27002"},
    {_id: 1, host: "shard2.2:27002"},
    {_id: 2, host: "shard2.3:27002", arbiterOnly:true}
]}
rs.initiate(config)
rs.status()
```

### Shard3
```bash
docker exec -it mongo_shard3.1 mongosh 127.0.0.1:27003
```
## configure shard3 nodes as a replicaSet
```javascript
config = {_id: "shard3", members: [
    {_id: 0, host: "shard3.1:27003"},
    {_id: 1, host: "shard3.2:27003"},
    {_id: 2, host: "shard3.3:27003", arbiterOnly:true}
  ]
}
rs.initiate(config)
rs.status()
```

## 配置管理员
```bash
docker exec -it mongo_mongos mongosh 127.0.0.1:20000
```

```javascript
use admin
db.createUser(
  {
    user: "admin",
    pwd: "admin",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
 db.grantRolesToUser(
  'admin',
     [
         { role: 'readWrite', db: 'timed-box-journal' },
         { role: 'dbAdmin', db: 'timed-box-journal' },
         { role: 'clusterAdmin', db: 'admin' }
     ]
 )

## 认证完成集群配置 
db.auth('admin', 'admin')
db.adminCommand(
  {
    setDefaultRWConcern : 1,
    defaultReadConcern: { level: "local" },
    defaultWriteConcern: { w: "majority", wtimeout: 5000 }
  }
)
sh.addShard("shard1/shard1.1:27001,shard1.2:27001,shard1.3:27001")
sh.addShard("shard2/shard2.1:27002,shard2.2:27002,shard2.3:27002")
sh.addShard("shard3/shard3.1:27003,shard3.2:27003,shard3.3:27003")
sh.enableSharding("timed-box-journal")
sh.shardCollection("timed-box-journal.akka_persistence_journal", {pid: 1})
```

## 使用业务库
```javascript
use timed-box-journal
```


## Sharding

