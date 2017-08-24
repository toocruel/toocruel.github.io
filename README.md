#图数据库Neo4j实现人脉推荐——二度人脉
本文来源：[http://blog.csdn.net/LoveJavaYDJ/article/details/53491346](http://blog.csdn.net/LoveJavaYDJ/article/details/53491346)

##一、业务需求：
通过现有系统“好友关系”和“用户通讯录”数据，实现人脉推荐——二度人脉….六度人脉
##二、技术实现分析：
关系数据库（深度关联表,算死人）

图数据库（天然图关系，选择Neo4j）

##三、业务实体模型及规模 ：
实体——顶点：用户（数百万）、手机号码（数亿）

关系——边：好友双向关系（数亿）、手机属于某个注册用户（数百万）、用户拥有手机号码（数亿）

事例(Neo4j中只存储关系相关数据)：

```
CREATE (p1:Person {uid:122622,identityType:1,personIUCode:'0101'})
CREATE (p2:Person {uid:122623,identityType:1,personIUCode:'0201'})
CREATE (p3:Person {uid:122624,identityType:2,personIUCode:'0301'})

CREATE (m1:Mobile {num:'1580084xxxx'})
CREATE (m2:Mobile {num:'1680084xxxx'})
CREATE (m3:Mobile {num:'1780084xxxx'})


CREATE
  /**p1分别和p2、p3互为双向好友*/ 
  (p1)-[:Fellow]->(p2),
  (p2)-[:Fellow]->(p1),

  (p1)-[:Fellow]->(p3),
  (p3)-[:Fellow]->(p1),

  /**用户p1通讯录中有m1、m2、m3三个联系人*/
  (p1)-[:Has]->(m1),
  (p1)-[:Has]->(m2),
  (p1)-[:Has]->(m3),

  (p2)-[:Has]->(m3),

  /**用户p1使用手机m1注册*/
  (m1)-[:Belong]->(p1) 
```

##四、实施方案&计划：
- 第一数据：

1. 找好服务器资源&安装配置好Neo4j数据库(24核64G内存，Neo4j配置20G)
2. 从mysql、mongodb中导出指定格式的用户、好友关系、通讯录CSV文件数据（导出所需时间？）
3. 导入CSV数据进入Neo4j数据库（导入时间？）

- 第二研发：

1. 开发、部署好Neo4j独立应用服务GraphServer
   - 服务1——addUser：新增用户Node
   * 服务2——updateUserIUCode：修改Neo4j中用户行业
   - 服务3——addFriend：新增用户好友关系
   * 服务4——delFriend：删除好友关系
   * 服务5——getFriendOfFriend：好友的好友
   * 服务6——synContactList2Neo4j：同步通讯录
2. 业务APP服务端应用接入GraphServer
   * 修改1：用户保存profile时，调用GraphServer.addUser()
   * 修改2：用户修改行业时，调用GraphServer.updateUserIUCode()
   * 修改3：用户同意添加好友时，调用GraphServer.addFriend()
   * 修改4：用户删除好友时，调用GraphServer.delFriend()，同时删除Reids缓存中推荐的此UID
   * 修改5：增加人脉推荐（优先查询Redis缓存,如有随机获取10；如没或少于10则调用GraphServer.getFriendOfFriend()获取数据且更新保存到Redis）

##五、导入数据：
- 用户数据（Node）：

```
[root@docker-vm yidejun]# head uidNew.csv 
uid:ID(user-ID),identityType,personIUCode,:LABEL
100002,5,"0309",Person
100004,1,"0105",Person
100025,4,"0111",Person
100001,4,"0101",Person
100055,7,"0101",Person
100058,3,"0521",Person
100065,0,"0116",Person
100069,9,"0116",Person
100400,0,"9921",Person
```

- 手机数据（Node）：

```
[root@docker-vm yidejun]# head mobile3_20161124.csv
num:ID(mobile-ID),:LABEL
"1309957XXXX",M
"1319509XXXX",M
"1323959XXXX",M
"1340950XXXX",M
"1340951XXXX",M
"1346950XXXX",M
"1346951XXXX",M
```

- 好友关系数据（Relationship）：

```
[root@docker-vm yidejun]# head dejun.csv
:START_ID(user-ID),:END_ID(user-ID),:TYPE
111360,109378,F
111360,106615,F
125952,119734,F
125952,119737,F
125952,119735,F
125952,109378,F
125952,118472,F
126208,126003,F
126208,110944,F
```

- 手机归属数据（Relationship）：

```
[root@docker-vm yidejun]# head mobile-belong-user.csv
:START_ID(mobile-ID),:END_ID(user-ID),:TYPE
"1391792XXXX",100002,Belong
"1399999XXXX",100004,Belong
"1860215XXXX",100025,Belong
"1502134XXXX",100001,Belong
"1331010XXXX",100055,Belong
```

- 通讯录被拥有数据（Relationship）：

```
[root@docker-vm yidejun]# head data4.csv
:START_ID(user-ID),:END_ID(mobile-ID),:TYPE
10000001,"1309957XXXX",H
10000001,"1319509XXXX",H
10000001,"1323959XXXX",H
10000001,"1340950XXXX",H
10000001,"1340951XXXX",H
10000002,"1346950XXXX",H
10000002,"1346951XXXX",H
```

- 导入脚本：

```
./bin/neo4j-import --into /usr/local/neo4j-community-3.0.7/data/databases/my.db --nodes /home/yidejun/uidNew.csv --nodes /home/yidejun/mobile3_20161124.csv  --relationships /home/yidejun/dejun.csv --relationships /home/yidejun/mobile-belong-user.csv --relationships /home/yidejun/data4.csv --skip-bad-relationships=true --stacktrace --bad-tolerance=50000 --skip-duplicate-nodes=true
```
![](images/neo4j-import.png)

- 导入效果（导入后启动Neo4j服务器，浏览器访问）:
![](images/neo4j-import-finish.png)

##六、数据验证 ：
1. 节点总数量

```
start n=node(*) return count(*)
```

2. 创建节点（保持uid唯一）

````
MERGE (n:Person {uid:'122'}) RETURN n;
start n=node:node_auto_index(uid='122') with count(*) as exists
where exists=0 create (n:Person {uid='122'}) return n;
````

3. 创建节点&关系

````
MERGE (one:Person {uid:'1222'}),(two:Person {uid:'1000'}) with one,two CREATE one-[:F]->two;
````

4. 删除关系（双向——不管什么关系）
````
Match (x:Person{uid:'1222'})-[r:F]-(y:Person{uid:'1223'}) delete  r;
````

5. 计算无关系的节点数
````
MATCH (player) WHERE NOT (player)-[:F]-() RETURN count(*)
````

6. 修改属性(批量修改参考http://stackoverflow.com/questions/38149000/neo4j-batch-update-of-data)

````
match (u:Person {uid:'122622'}) set u.personIUCode='0205' return u
````

7. 我的通讯录联系人在系统中注册，但和我还不是好友（我有他的手机号码）

````
match (n:Person {uid:1222})-[:H]->(m)-[:Belong]->(b) where not (n)-[:F]-(b)   return b.uid
````

8. 拥有我的号码，但不是好友（他有我的手机号码）
````
match (n:Person {uid:1222})<-[:Belong]-(m)<-[:H]-(b) where not (n)-[:F]-(b)   return b.uid
````

9. 可能认识（我们都有同一个手机号码）

````
match (n:Person {uid:122})-[:H]->(m)<-[:H]-(b) with m,b where not (n)-[:F]-(b)   return DISTINCT  b
````

10. 有N个共同好友

````
MATCH (a:Person {uid:{1}})-[:F]->(friend)<-[:F]-(b:Person) RETURN b.uid AS uid,COUNT(*) AS comCount,4 AS type ORDER BY comCount DESC SKIP {2} LIMIT {3}
````

11. 有N个共同好友列表

````
match (a:Person {uid:'122622'})-[:F]->(friend)<-[:F]-(b:Person {uid:'100004'}) return friend
````

12. 好友的好友中交集最多的

````
Match (user:Person {uid:{1}}) match (user)-[:F]->(friend:Person {personIUCode:{2}})-[:F]->(fof:Person {personIUCode:{2}}) with user,fof OPTIONAL Match (user)-[r]-(fof) with fof,r where r is null  RETURN fof.uid as uid, COUNT(*) as count ORDER BY COUNT(*) DESC, fof.uid limit 20
````

13. 最短路径问题

````
Match(a:Person {uid:122}) Match(b:Person {uid:100})  p= shortpath((a) -[:F*1..2]->(b)) return a,b,p
MATCH (n:Person { uid:122 })-[:F*1..2]-(neighbors) RETURN n, collect(DISTINCT neighbors)
MATCH p=allShortestPaths((a:Person { uid: 1222})-[:F*..4]-(b:Person { uid: 123})) RETURN p
````

##七、踩过的坑 ：
1. 服务器句柄数设置:

````
Starting Neo4j.
WARNING: Max 1024 open files allowed, minimum of 40000 recommended. See the Neo4j manual.
````

2. 导入数据——明确指定相关ID:
![](images/neo4j-import-q1.png)

3. 导入数据——重要参数设置，否则莫名毫无提示的停止导入 :

````
–skip-bad-relationships=true 
–stacktrace 
–bad-tolerance=50000 
–skip-duplicate-nodes=true
````

4. 版本兼容之OPTION MATCH 正确使用，参考https://dzone.com/articles/new-neo4j-optional LOAD CSV超级慢

````
LOAD CSV WITH HEADERS FROM "file:/home/yidejun/account.csv" AS line RETURN COUNT(*);

LOAD CSV WITH HEADERS FROM "file:/home/yidejun/account.csv" AS line RETURN line LIMIT 5;

LOAD CSV WITH HEADERS FROM "file:/home/yidejun/account.csv" AS line FIELDTERMINATOR '\t' RETURN line.account LIMIT 5;

LOAD CSV WITH HEADERS FROM "file:/home/yidejun/account.csv" AS line FIELDTERMINATOR '\t' RETURN line.uid LIMIT 5;
USING PERIODIC COMMIT 100 LOAD CSV WITH HEADERS FROM "file:/home/yidejun/account.csv" AS line FIELDTERMINATOR '\t' WITH line CREATE (account:Account {account:line.account}) WITH line,account  MATCH (u:Person {uid:line.uid}) WITH account,u CREATE (account)-[:Belong]->(u);
````

##【附】主要参考：

Cypher语法手册：[http://neo4j.com/docs/cypher-refcard/3.0/](http://neo4j.com/docs/cypher-refcard/3.0/)

导入工具：[http://neo4j.com/docs/operations-manual/current/tutorial/import-tool/](http://neo4j.com/docs/operations-manual/current/tutorial/import-tool/)

##【附】JAVA代码：
Git(开源)：[https://github.com/Aresyi/Neo4jForRec](https://github.com/Aresyi/Neo4jForRec)