### 使用sshtunnel连接内网mongo

#### 跳板机参数

```python
jumpServerIP = 'test'
jumpServerUsername = 'root'
jumpServerPassword = 'password'
```

#### mongodb参数

```python
# 密码中若含有特殊字符@，用%40代替
mongoIP = '10.0.5.10'
mongoUsername = 'mongouser'
mongoPassword = 'test%402018'
mongoDatabase = 'dbName' # 默认数据库
mongoCollection = 'collName'
```

#### 建立sshtunnel并读写mongo

```python
def writeMongoClinetInfo():

    with SSHTunnelForwarder(
        ssh_address_or_host=jumpServerIP,
        ssh_password=jumpServerPassword,
        ssh_username=jumpServerUsername,
        remote_bind_address=(mongoIP, 27017),
        local_bind_address=('127.0.0.1',22222)
    ) as server:
        server.start()
        # 这里注意uri中的mongo ip及port一定是在ssh tunnel中绑定的本地地址(local_bind_address)
        uri = "mongodb://%s:%s@%s/%s?authMechanism=SCRAM-SHA-1&authSource=admin" % (mongoUsername, mongoPassword, '127.0.0.1:22222', mongoDatabase)
        print(uri)
        client = pymongo.MongoClient(uri)
        # 也可以通过下面的方式认证mongo
        # client = pymongo.MongoClient('127.0.0.1', server.local_bind_port)
        # client.iov_account.authenticate(mongoUsername, mongoPassword, mechanism='SCRAM-SHA-1', source='admin')
        # 选取db
        db = client[mongoDatabase]
        # 选取collection
        collection = db[mongoCollection]
        try:
            # 插入数据，根据主键定义可能会抛出异常，当抛出异常的时候更新对应数据
            result = collection.insert(formData)
            print("insert success, result:",result)
        except pymongo.errors.DuplicateKeyError:
            condition = {'_id':clientId}
            result = collection.update(condition,formData)
            print("update success, result:", result)
        # 关闭隧道
        server.close()
        server.stop()

if __name__ == '__main__':
	writeMongoClinetInfo()
```

#### 使用mongoexport和mongoimport导出导入数据

##### mongoexport

```shell
./mongoexport --host <mongo ip> --port 27017 --username <用户名> --password <密码> --authenticationDatabase admin --db <数据库名> --collection <集合名> --csv --out <导出文件名> --fields <字段名>
```

一个例子：

```shell
# 将数据库myDatabase中myCollection的表中的userId,nation等字段导出到文件use.csv中
# 注意，导出到csv文件时必须制定要导出的field
./mongoexport --host 127.0.0.1 --port 27017 --username mongouser --password myPassword --authenticationDatabase admin --db myDatabase --collection myCollection --csv --out /root/horizonliu/formal/20190306/user.csv --fields 'userId,nation,phone,email,wxOpenId,name,sex'
```

#### mongoimport

```shell
# 将文件中的内容导入到mongo对应的collection中
./mongoimport <mongo ip> --port 27017 --username <用户名> --password <密码> --authenticationDatabase admin --db <数据库名> --collection <集合名> --file client.csv
```