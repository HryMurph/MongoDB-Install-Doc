# MongoDB、mongo-connector 与 Elasticsearch

本文档以实现 MongoDB 的副本集，并使用 mongo-connector 连接 Elasticsearch 为目的。测试用主机使用两台Ubuntu19.04 虚拟机，IP 地址分别为：192.168.11.131（主机 Primary）和192.168.11.128（从属机 Secondary）。MongoDB 若不配置副本集，则无法与 Elasticsearch 连接。

## MongoDB

### 安装 MongoDB

#### 导入公钥

Ubuntu 软件包管理器 apt（高级软件包工具）需要软件分销商的 GPG 密钥来确保软件包的一致性和真实性。 运行此命令将 MongoDB 密钥导入到您的服务器。

`sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 2930ADAE8CAF5059EE73BB4B58712A2291FA4AD5`

#### 创建源列表文件 MongoDB

使用以下命令在 /etc/apt/sources.list.d/ 中创建一个 MongoDB 列表文件：

`sudo echo "deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.6 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.6.list`

使用 xenial 版本好于 bionic 版本，后者在测试中发现会造成 apt-get update 404。

#### 更新存储库

使用 apt 命令更新存储库：

`sudo apt-get update`

#### 安装 MongoDB

`sudo apt-get install -y mongodb-org`

#### 启动 MongoDB 并将其添加为在启动时启动的服务

`sudo systemctl start mongod`
`sudo systemctl enable mongod`

#### 如果要停止 MongoDB 服务：

`sudo systemctl stop mongod`

### 配置 MongoDB

#### 配置数据库管理员

1. 打开 MongoDB Shell：

   `mongo`

2. 切换到 admin 数据库：

   `use admin`

3. 创建管理员用户：

   `db.createUser({user:"admin", pwd:"admin", roles:[{role:"userAdminAnyDatabase", db:"admin"}]})`

   创建成功时应有如下提示：

   ```json
   Successfully added user: {
   	"user" : "admin",
   	"roles" : [
   		{
   			"role" : "userAdminAnyDatabase",
   			"db" : "admin"
   		}
   	]
   }
   ```

#### 配置数据库外部访问与副本集

1. 退出 MongoDB Shell：

   `exit`

2. 使用 root 权限编辑配置文件 /etc/mongod.conf：

   ```
   storage:
     dbPath: /var/lib/mongodb
     journal:
       enabled: true
   
   systemLog:
     destination: file
     logAppend: true
     path: /var/log/mongodb/mongod.log
   
   # network interfaces
   net:
     port: 27017
     bindIp: 0.0.0.0
   
   replication:
     oplogSizeMB: 2048
     replSetName: rs0
   ```

   重点关注的字段为 bindIp 与 replication。并且检查 dbPath 和 logPath 对应的文件/文件夹是否存在，不存在应用 root 权限创建。

3. 保存后重启 MongoDB 服务：

   `sudo systemctl restart mongod`

4. 重新进入 MongoDB Shell：

   `mongo`

   这里需要注意，如果无法进入，提示连接被拒，那应该是服务没有正确启动导致的，应查看 log 文件（配置文件里面设置的）分析错误原因，大概率是配置文件没有编辑正确，不符合规则。

6. 初始化副本集：

   `rs.initiate({_id: 'rs0', members: [{_id:0, host: '192.168.11.131:27017'},{_id: 1, host: '192.168.11.128:27017'}]});`

   初始化成功时应有如下提示：

   ```json
   {
   	"ok" : 1,
   	"operationTime" : Timestamp((1583899078, 1),
   	"$clusterTime" : {
   		"clusterTime" : Timestamp((1583899078, 1),
   		"signature" : {
   			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
   			"keyId" : NumberLong(0)
   		}
   	}
   }
   ```

   重新进入 MongoDB Shell 会出现提示：rs0:PRIMARY> （rs0:SECONDARY> ）

7. 验证副本集配置：

   `rs.conf()`

   可得到如下 json 返回：

   ```json
   {
   	"_id" : "rs0",
   	"version" : 1,
   	"protocolVersion" : NumberLong(1),
   	"members" : [
   		{
   			"_id" : 0,
   			"host" : "192.168.11.131:27017",
   			"arbiterOnly" : false,
   			"buildIndexes" : true,
   			"hidden" : false,
   			"priority" : 1,
   			"tags" : {
   				
   			},
   			"slaveDelay" : NumberLong(0),
   			"votes" : 1
   		},
   		{
   			"_id" : 1,
   			"host" : "192.168.11.128:27017",
   			"arbiterOnly" : false,
   			"buildIndexes" : true,
   			"hidden" : false,
   			"priority" : 1,
   			"tags" : {
   				
   			},
   			"slaveDelay" : NumberLong(0),
   			"votes" : 1
   		}
   	],
   	"settings" : {
   		"chainingAllowed" : true,
   		"heartbeatIntervalMillis" : 2000,
   		"heartbeatTimeoutSecs" : 10,
   		"electionTimeoutMillis" : 10000,
   		"catchUpTimeoutMillis" : -1,
   		"catchUpTakeoverDelayMillis" : 30000,
   		"getLastErrorModes" : {
   			
   		},
   		"getLastErrorDefaults" : {
   			"w" : 1,
   			"wtimeout" : 0
   		},
   		"replicaSetId" : ObjectId("5e6855e4fd539f6e02a8bc14")
   	}
   }
   ```

8. 验证副本集状态：

   `rs.status()`

   可得到如下 json 返回：

   ```json
   {
   	"set" : "rs0",
   	"date" : ISODate("2020-03-11T03:58:02.281Z"),
   	"myState" : 1,
   	"term" : NumberLong(1),
   	"syncingTo" : "",
   	"syncSourceHost" : "",
   	"syncSourceId" : -1,
   	"heartbeatIntervalMillis" : NumberLong(2000),
   	"optimes" : {
   		"lastCommittedOpTime" : {
   			"ts" : Timestamp(1583899078, 1),
   			"t" : NumberLong(1)
   		},
   		"readConcernMajorityOpTime" : {
   			"ts" : Timestamp(1583899078, 1),
   			"t" : NumberLong(1)
   		},
   		"appliedOpTime" : {
   			"ts" : Timestamp(1583899078, 1),
   			"t" : NumberLong(1)
   		},
   		"durableOpTime" : {
   			"ts" : Timestamp(1583899078, 1),
   			"t" : NumberLong(1)
   		}
   	},
   	"members" : [
   		{
   			"_id" : 0,
   			"name" : "192.168.11.131:27017",
   			"health" : 1,
   			"state" : 1,
   			"stateStr" : "PRIMARY",
   			"uptime" : 3737,
   			"optime" : {
   				"ts" : Timestamp(1583899078, 1),
   				"t" : NumberLong(1)
   			},
   			"optimeDate" : ISODate("2020-03-11T03:57:58Z"),
   			"syncingTo" : "",
   			"syncSourceHost" : "",
   			"syncSourceId" : -1,
   			"infoMessage" : "",
   			"electionTime" : Timestamp(1583896046, 1),
   			"electionDate" : ISODate("2020-03-11T03:07:26Z"),
   			"configVersion" : 1,
   			"self" : true,
   			"lastHeartbeatMessage" : ""
   		},
   		{
   			"_id" : 1,
   			"name" : "192.168.11.128:27017",
   			"health" : 1,
   			"state" : 2,
   			"stateStr" : "SECONDARY",
   			"uptime" : 3045,
   			"optime" : {
   				"ts" : Timestamp(1583899078, 1),
   				"t" : NumberLong(1)
   			},
   			"optimeDurable" : {
   				"ts" : Timestamp(1583899078, 1),
   				"t" : NumberLong(1)
   			},
   			"optimeDate" : ISODate("2020-03-11T03:57:58Z"),
   			"optimeDurableDate" : ISODate("2020-03-11T03:57:58Z"),
   			"lastHeartbeat" : ISODate("2020-03-11T03:58:01.656Z"),
   			"lastHeartbeatRecv" : ISODate("2020-03-11T03:58:00.634Z"),
   			"pingMs" : NumberLong(0),
   			"lastHeartbeatMessage" : "",
   			"syncingTo" : "192.168.11.131:27017",
   			"syncSourceHost" : "192.168.11.131:27017",
   			"syncSourceId" : 0,
   			"infoMessage" : "",
   			"configVersion" : 1
   		}
   	],
   	"ok" : 1,
   	"operationTime" : Timestamp(1583899078, 1),
   	"$clusterTime" : {
   		"clusterTime" : Timestamp(1583899078, 1),
   		"signature" : {
   			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
   			"keyId" : NumberLong(0)
   		}
   	}
   }
   ```

9. 副本集复制测试：

   在主机 PRIMARY 上插入10000条客户数据：

   `for(var i=0;i<10000;i++){db.customer.insert({"name":"user"+i})}`

   成功时应返回：

   `WriteResult({ "nInserted" : 1 })`

   查询客户数据量：

   `db.customer.count()`

   应正确返回：

   `10000`

   此时数据应该正常同步到从属机 SECONDARY 上，输入：

   `rs.slaveOk()`

   `db.customer.count()`

   应正确返回：

   `10000`



## mongo-connector 与 Elasticsearch

#### 安装 mongo-connector

`pip install mongo-connector`

`pip install elastic2-doc-manager elasticsearch`

`pip install 'mongo-connector[elastic5]'`

#### 安装与配置 Elasticsearch

`wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.16.tar.gz`

`sudo tar -C /usr/lib/ -zxf elasticsearch-5.6.16.tar.gz`

`sudo mv /usr/lib/elasticsearch-5.6.16 /usr/lib/elasticsearch`

创建一个新用户 es 并修改其最大文件数：

`sudo useradd es`

`sudo passwd es`

`es`（用户 es 的密码为 es）

`sudo vim /etc/security/limits.conf`

```
es soft nofile 65536
es hard nofile 65536
es soft noproc 4096
es hard noproc 4096
```

给用户 es 授 elasticsearch 访问权限：

`sudo chown es /usr/lib/elasticsearch -R`

修改配置文件：

`sudo vim /usr/lib/elasticsearch/conf/elasticsearch.yml`

```
network.host: 0.0.0.0
```

登录到 es，进入 elasticsearch 文件夹并执行后台运行：

`cd /usr/lib/elasticsearch/bin`

`./elasticsearch -d`

新创建的用户使用的 shell 为 sh，界面较简陋，操作不友好，建议将新用户的 shell 更换为 bash。

实际操作中，为避免频繁查看 log 来排除报错，可先在终端中运行（不加后面的参数 -d），确认正常执行后再使用后台运行。

成功开启 Elasticsearch 时，使用浏览器访问 http://192.168.11.131:9200 应返回 json 数据：

```json
{
  "name" : "9s***uB",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Gm******************3Q",
  "version" : {
    "number" : "5.6.16",
    "build_hash" : "3a***d1",
    "build_date" : "2019-03-13T15:33:36.565Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.1"
  },
  "tagline" : "You Know, for Search"
}
```

#### 配置 mongo-connector

以蓝鲸 CMDB 的官方部署引导举例，配置前，需要在 MongoDB 内

为便于维护，在 mongo-connector 包内新建一个 config.json 文件：

```json
{
    "__comment__": "Configuration options starting with '__' are disabled",
    "__comment__": "To enable them, remove the preceding '__'",

    "mainAddress": "127.0.0.1:27017",
    "oplogFile": "/var/log/mongo-connector/oplog.timestamp",
    "noDump": false,
    "batchSize": -1,
    "verbosity": 3,
    "continueOnError": true,

    "logging": {
        "type": "file",
        "filename": "/var/log/mongo-connector/mongo-connector.log",
        "format": "%(asctime)s [%(levelname)s] %(name)s:%(lineno)d - %(message)s",
        "rotationWhen": "D",
        "rotationInterval": 1,
        "rotationBackups": 10,

        "__type": "syslog",
        "__host": "localhost:514"
    },

    "__authentication": {
        "adminUsername": "cc",
        "password": "cc",
        "__passwordFile": "mongo-connector.pwd"
    },

    "__fields": ["field1", "field2", "field3"],
    
    "exclude_fields": ["create_time", "last_time"],

    "namespaces": {
        "cmdb.cc_HostBase": true,
        "cmdb.cc_ObjectBase": true,
        "cmdb.cc_ObjDes": true,
        "cmdb.cc_ApplicationBase": true,
        "cmdb.cc_OperationLog": false
    },

    "docManagers": [
        {
            "docManager": "elastic2_doc_manager",
            "targetURL": "127.0.0.1:9200",
            "__bulkSize": 1000,
            "uniqueKey": "_id",
            "autoCommitInterval": 0
        }
    ]
}
```

运行mongo-connector：

`sudo mongo-connector -c config.json`

成功运行时：

```
/usr/local/lib/python2.7/dist-packages/mongo_connector/compat.py:30: UserWarning: Python 2 support is deprecated and pending removal. Please run mongo-connector on Python 3. See https://github.com/yougov/mongo-connector/issues/829 for more details or to post concerns.
  "Python 2 support is deprecated and pending removal. Please "
Logging to /var/log/mongo-connector/mongo-connector.log.

```

以上的警告代表 Python2 即将结束技术支持，无需理会。
