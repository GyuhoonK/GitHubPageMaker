---
layout: post
current: post
cover: assets/built/images/hadoop/distcp.png
navigation: True
title: hadoop distcp
date: 2022-01-20 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

hadoop distcp 명령어

# `distcp`

`distcp` 명령어는 효율적인 병렬 처리를 통해 파일을 복사합니다. 아래와 같은 명령어로 실행합니다.

```shell
# 파일 복사
[gyuhoonk@namenode1~]$ hadoop distcp file1 file2
# hadoop fs -cp file1 file2 와 같은 결과이지만
# distcp를 이용하는 경우에는 '병렬 처리'하여 복사함

# 디렉토리 복사
[gyuhoonk@namenode1~]$ hadoop distcp dir1 dir2
```

`discp`는 내부적으로는 `hadoop fs -cp`명령을 클라이언트가 직접 수행하는 방식이기 때문에 큰 파일의 복사에 더 적합합니다. 

# options
[DistCp Guide](https://hadoop.apache.org/docs/stable/hadoop-distcp/DistCp.html)
```shell
# overwrite
[gyuhoonk@namenode1~]$ hdfs dfs ls 
>>> dir1 dir2


[gyuhoonk@namenode1~]$ hadoop distcp dir1 dir2
# 이미 dir2가 존재하고 있으므로, dir2의 하위에 dir1을 복사함


[gyuhoonk@namenode1~]$ hadoop distcp -overwrite dir1 dir2
# overwrite 옵션이 있으므로 dir2에 dir1을 덮어씌움



# update
# 디렉토리 내에 변경이 있는 파일들만 복사하여 동기화시킵니다.
[gyuhoonk@namenode1~]$ hadoop distcp -update dir1 dir2
# dir1 내 파일과 dir2 내 파일을 비교하여 변경된 부분만 dir2에 동기화시킴



# delete
# 원본 경로에는 존재하지 않고, 타겟 경로에만 존재하는 파일들을 지우도록 하는 옵션입니다.
[gyuhoonk@namenode1~]$ hadoop distcp -delete dir1 dir2
# dir1, dir2를 비교하여 dir2에만 존재하는 파일들은 삭제한 뒤 distcp를 수행



# m
# 몇 개의 mapper를 사용할지 결정합니다. 
# 기본적으로 distcp는 Map-Reduce Job으로 구현되어있으며 
# 클러스터 전반에 걸쳐 병렬로 수행되는 Map Task를 이용하여 복사 작업을 수행합니다.
# 단, Reducer를 사용하지 않으며 각 파일은 Mapper에서 복사합니다. 
# 이 때 bucketing을 통해 각 Mapper에 거의 같은 양의 데이터를 제공하고자 합니다. 
# 기본값으로 최대 20개의 Mapper가 사용됩니다.
[gyuhoonk@namenode1~]$ hadoop distcp -m 100 file1 file2
# 100개 Mapper를 사용하여 distcp 수행



# p
#복제 시 파일의 권한, 블록 사이즈 등 파일 속성 정보를 보전하려는 경우에 사용됩니다. 
[gyuhoonk@namenode1~]$ hadoop distcp -p file1 file2
```



# 다른 클러스터(namenode)간 복사

```shell
[gyuhoonk@namenode1~]$ hadoop distcp webhdfs://namenode1:14000/foo webhdfs://namenode2:14000/foo
```

위처럼 namenode1에서 namenode2로 파일을 복사할 수 있습니다. 이 경우에 webHDFS 프로토콜을 이용합니다. webHDFS 프로토콜 대신에 HttpFs 프록시 방식으로 distcp의 소스 혹은 타깃을 변경할 수도 있다. HttpFs 프록시 방식은 방화벽, 대역폭 설정을 할 수 있다는 장점이 있습니다.

참고로 HDFS에 접근 가능한 인터페이스는 아래와 같습니다.

- webHDFS

HDFS는 REST API를 이용하여 파일을 조회, 생성, 수정, 삭제(CRUD)를 지원합니다. 이러한 기능을 제공하는 API가 webHDFS이며  FileSystem API입니다. 해당 프로토콜은 Java FileSystem Class에 작성되어 있습니다. 그리고 이를 바탕으로 다른 FileSystem을 이용할 수도 있습니다. 

- Http

JAVA로 작성되지 않은 애플리케이션이 HDFS에 접근할 수 있도록 WebHDFS 프로토콜을 기반으로 하여 HTTP REST API를 사용할 수 있습니다. 

- C - libhdfs library

JAVA FileSystem Class를 모방하여 작성된 라이브러리입니다. 모든 HDFS(Local, S3 등)에 접근 가능합니다. JNI(Java Native Interface)를 사용하여 JAVA FileSystem Class를 호출합니다. 

- NFS

NFSv3 게이트웨이를 이용하여 로컬 클라이언트 파일시스템에 HDFS를 마운트할 수 있습니다. 또한, ls, cat과 같은 unix 명령어를 이용할 수 있습니다. 참고로, HDFS는 파일의 끝에만 쓰기를 허용하므로 파일에 추가하는 작업은 가능하지만, 파일의 임의 위치에 있는 데이터를 수정하는 것은 지원하지 않습니다.

- FUSE(Filesystem in Userspace)

사용자 공간과 유닉스 파일 시스템을 통합한 파일 시스템을 지원합니다. 하둡의 Fuse-DFS contrib 모듈은 표준 로컬 파일시스템에 HDFS를 마운트할 수 있는 기능을 제공합니다. Fuse-DFS는 C로 작성된 libhdfs로 HDFS 인터페이스를 구현했습니다. 참고로 NFS가 Fuse-DFS보다 안전하며 선호되는 방법입니다.

# 클러스터간 테이블 이동

예를 들어, namenode1에 아래와 같은 테이블이 있다고 가정해보겠습니다.

```shell
[gyuhoonk@namenode1~]$ hive -e 'SELECT * FROM default.table_1'
+-------+---------+
| col1  |  col2   |
+-------+---------+
| a     | apple   |
| b     | banana  |
| c     | car     |
+-------+---------+
```

namenode2로 default.table_1을 이동하기 위해서는 아래와 같은 과정이 필요합니다.

1. `/user/hive/warehouse/default.db/table_1` 디렉토리 이동 

```shell
# namenode1에서 실행
[gyuhoonk@namenode1~]$ hadoop distcp -overwrite webhdfs://namenode1:14000/user/hive/warehouse/default.db/table_1 webhdfs://namenode2:14000/user/hive/warehouse/default.db/table_1

#namenode2에서 distcp 결과 확인
[gyuhoonk@namenode2~]$ hdfs dfs -ls /user/hive/warehouse/default.db/table_1
-rw-r--r--+  3 gyuhoonK hive          0 2022-01-20 10:04 /user/hive/warehouse/defulat.db/table_1/_SUCCESS
-rw-r--r--+  3 gyuhoonK hive        340 2022-01-20 10:04 /user/hive/warehouse/defulat.db/table_1/part-00000-7628a86e-50d7-4e64-a35d-435ba6943156-c000.snappy.parquet
-rw-r--r--+  3 gyuhoonK hive        544 2022-01-20 10:04 /user/hive/warehouse/defulat.db/table_1/part-00166-7628a86e-50d7-4e64-a35d-435ba6943156-c000.snappy.parquet
-rw-r--r--+  3 gyuhoonK hive        549 2022-01-20 10:04 /user/hive/warehouse/defulat.db/table_1/part-00333-7628a86e-50d7-4e64-a35d-435ba6943156-c000.snappy.parquet
-rw-r--r--+  3 gyuhoonK hive        534 2022-01-20 10:04 /user/hive/warehouse/defulat.db/table_1/part-00499-7628a86e-50d7-4e64-a35d-435ba6943156-c000.snappy.parquet
```

2.  create statement 확인

```shell
[gyuhoonk@namenode1~]$ hive -e 'SHOW CREATE TABLE default.table_1'
```
```sql
+----------------------------------------------------+
|                   createtab_stmt                   |
+----------------------------------------------------+
| CREATE EXTERNAL TABLE `default.table_1`(           |
|   `col1` string,                                   |
|   `col2` string)                                   |
| ROW FORMAT SERDE                                   |
|   'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'  |
| WITH SERDEPROPERTIES (                             |
|   'path'='hdfs://nameservice1/user/hive/warehouse/default.db/table_1')                                          |
| STORED AS INPUTFORMAT                              |
|   'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat'  |
| OUTPUTFORMAT                                       |
|   'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat' |
| LOCATION                                           |
|   'hdfs://nameservice1/user/hive/warehouse/default.db/table_1' |
| TBLPROPERTIES (                                    |
|   'bucketing_version'='2',                         |
|   'spark.sql.create.version'='2.3.1.3.0.1.0-187',  |
|   'spark.sql.sources.provider'='parquet',          |
|   'spark.sql.sources.schema.numParts'='1',         |
|   'spark.sql.sources.schema.part.0'='{"type":"struct","fields":[{"name":"col1","type":"string","nullable":true,"metadata":{}},{"name":"col2","type":"string","nullable":true,"metadata":{}}]}',  |
|   'transient_lastDdlTime'='1642640799')            |
+----------------------------------------------------+
```

아래와 같이 명령어를 조합하면 쿼리문만 파일로 저장할 수 있습니다.

```shell
[gyuhoonk@namenode1~]$ hive -e 'show create table default.table_1' | sed 's/|//g' | sed 's/+//g' | sed 's/createtab_stmt//g' | sed '/WARN/d' >> create.sql 2>/dev/null; echo ';' >> create.sql
```

3. `create.sql` 파일을 namenode2로 옮긴 뒤 namenode2에서 create statement 실행

```shell
# namenode1
[gyuhoonk@namenode1~]$ scp create.sql gyuhoonk@namenode2:/home/gyuhoonk/
# namenode2
[gyuhoonk@namenode2~]$ hive -f create.sql
```

4. Partitioned table인 경우에는 TABLE REPAIR 실행

```shell
[gyuhoonk@namenode2~]$ hive -e 'MSCK REPAIR TABLE defulat.table_1'
```

[참고]

하둡 완벽 가이드, 한빛미디어

[DistCp Guide](https://hadoop.apache.org/docs/r1.2.1/distcp.html)
