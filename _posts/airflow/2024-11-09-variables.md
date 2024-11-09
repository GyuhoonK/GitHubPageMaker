---
layout: post
current: post
cover: assets/built/images/airflow/airflow.png
navigation: True
title: Variables & Dag Parsing
date: 2024-11-08 22:30:00 +0900
tags: [airflow]
class: post-template
subclass: 'post tag-airflow'
author: GyuhoonK
---

Variables 사용 방법을 알아보고, Dag Parsing에서 그 이유를 살펴봅니다.


## Variables

### key value

### json serialize

### 암호화 (fernet_key)

## Best Practice 

### Good
#### execute() 내부에서는 자유롭게 사용

#### jinja template 사용하여 전달

### Bad

#### TOP LEVEL PYTHON 코드에서 get 메소드 사용

#### 대체 방법
- jinja
- task(execute) 내에서 get 메소드 사용

## Dag Parsing

### Top Level Python Code

### Dag Parsing
- Dag Serializtion의 Dag File Processing
- Airflow에서 사용하고 있는 MySQL(Metadata DataBase)와 연결되지 않는 것이 좋음

### Dag Running
- 실제로 실행
- Airflow에서 사용하고 있는 MySQL(Metadata DataBase)와 연결됨


## Variable Cache

[참고]  
[[Airflow] Airflow DAG Serialization](https://wookiist.dev/168)  
[The ins and outs of Airflow’s new Secrets Cache](https://medium.com/apache-airflow/the-ins-and-outs-of-airflows-new-secrets-cache-f7b9ec25ca1e)
