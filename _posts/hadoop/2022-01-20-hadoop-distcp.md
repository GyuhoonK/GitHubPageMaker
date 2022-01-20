---
layout: post
current: post
cover: assets/built/images/hadoop/spark-overwrite-partition.jpg
navigation: True
title: Overwrite Partition in Spark
date: 2022-01-20 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

hadoop distcp

# `distcp`

`distcp` 명령어는 효율적인 병렬 처리를 통해 파일을 복사합니다. 아래와 같은 명령어로 실행합니다.

```shell
# 파일 복사
% hadoop distcp file1 file2
# hadoop fs -cp file1 file2 와 같은 결과이지만
# distcp를 이용하는 경우에는 '병렬 처리'하여 복사함

# 디렉토리 복사
% hadoop distcp dir1 dir2
```

`discp`는 내부적으로는 `hadoop fs -cp`명령을 클라이언트가 직접 수행하는 방식이기 때문에 큰 파일의 복사에 더 적합합니다. 

# options

- overwrite

```shell
% hdfs dfs ls 
>>> dir1 dir2

% hadoop distcp dir1 dir2
# 이미 dir2가 존재하고 있으므로, dir2의 하위에 dir1을 복사함

% hadoop distcp -overwrite dir1 dir2
# overwrite 옵션이 있으므로 dir2에 dir1을 덮어씌움

```

- update 

디렉토리 내에 변경이 있는 파일들만 복사하여 동기화시킵니다.

```shell
% hadoop distcp -update dir1 dir2
# dir1 내 파일과 dir2 내 파일을 비교하여 변경된 부분만 dir2에 동기화시킴
```

- delete

원본 경로에는 존재하지 않고, 타겟 경로에만 존재하는 파일들을 지우도록 하는 옵션입니다.

```shell
% hadoop distcp -delete dir1 dir2
# dir1, dir2를 비교하여 dir2에만 존재하는 파일들은 삭제한 뒤 distcp를 수행
```

- m

몇 개의 mapper를 사용할지 결정합니다. 기본적으로 distcp는 Map-Reduce Job으로 구현되어있으며 클러스터 전반에 걸쳐 병렬로 수행되는 Map Task를 이용하여 복사 작업을 수행합니다. 단, Reducer를 사용하지 않으며 각 파일은 Mapper에서 복사합니다. 이 때 bucketing을 통해 각 Mapper에 거의 같은 양의 데이터를 제공하고자 합니다. 기본값으로 최대 20개의 Mapper가 사용됩니다.

```shell
% hadoop distcp -m 100 file1 file2
# 100개 Mapper를 사용하여 distcp 수행
```

- p

파일의 권한, 블록 사이즈 등 파일 속성 정보를 복제 시 보전하려는 경우에 사용됩니다. 



# webhdfs 프로토콜

```shell
% hadoop distcp webhdfs://namenode1:50070/foo webhdfs://namenode2:50070/foo
```



[참고]

[Overwrite specific partitions in spark dataframe write method](https://stackoverflow.com/questions/38487667/overwrite-specific-partitions-in-spark-dataframe-write-method)
