---
layout: post
current: post
cover: assets/built/images/database/preparedstatement.png
navigation: True
title: prepared statement
date: 2023-10-29 22:30:00 +0900
tags: [database]
use_math: True
class: post-template
subclass: 'post tag-database'
author: GyuhoonK
---

prepared statment

# prepared statment란?

## 개념
위키피디아의 설명을 먼저 살펴보겠습니다.

>프리페어드 스테이트먼트(prepared statement), 파라미터라이즈드 스테이트먼트(parameterized statement)는 데이터베이스 관리 시스템(DBMS)에서 동일하거나 비슷한 데이터베이스 문을 높은 효율성으로 반복적으로 실행하기 위해 사용되는 기능이다. 일반적으로 쿼리나 업데이트와 같은 SQL 문과 함께 사용되는 프리페어드 스테이트먼트는 템플릿의 형태를 취하며, 그 템플릿 안으로 특정한 상수값이 매 실행 때마다 대체된다.  
프리페어드 스테이트먼트의 일반적인 워크플로는 다음과 같다:  
1. 준비(Prepare): 먼저 애플리케이션은 문의 틀을 만들고 이를 DBMS로 보낸다. 특정값은 지정하지 않은 채로 남겨지며 이들은 "변수", "플레이스홀더", "바인드값"으로 부른다. (아래의 "?" 레이블 참고):  
  `INSERT INTO products (name, price) VALUES (?, ?);`  
2. 그 다음, DBMS는 문의 틀을 컴파일하며(최적화 및 변환) 아직 실행하지 않고 결과만 저장한다.  
3. 실행(Execute): 나중에 애플리케이션이 문 틀의 변수에 값(바인드)을 지정하면 DBMS는 (결과를 반환할 수도 있는) 문을 실행한다. 애플리케이션은 여러 값으로 원하는 횟수만큼 문을 실행할 수 있다. 위의 예에서 첫 번째 변수로 "bike"로, 두 번째 변수로 "10900"을 지정한다.

위키피디아에 적혀진 설명에서 핵심이 되는 단어는 두 가지인 것 같습니다. 

- 반복: 같은 쿼리를 반복 실행하는 경우에 높은 효율성을 위해 사용한다
- 준비: template(틀)을 미리 만들어 놓고, placeholder를 입력하여 보낸다.

주의할 점은 template과 placeholder를 전달받은 DBMS가 실행할 쿼리를 컴파일한다는 것입니다(어플리케이션에서 컴파일하여 그 결과를 전달하지 않는다, 어플리케이션은 template과 placeholder만 제출한다).

## 예시 (python)

이것도 위키피디아에 적절한 예시가 있어서 그대로 가져왔습니다. 

```python
import mysql.connector

conn = None
cursor = None

try:
    conn = mysql.connector.connect(database="mysql", user="root")
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS products (name VARCHAR(40), price INT)")
    params = [("bike", 10900),
              ("shoes", 7400),
              ("phone", 29500)]
    cursor.executemany("INSERT INTO products VALUES (%s, %s)", params)
    params = ("shoes",)
    cursor.execute("SELECT * FROM products WHERE name = %s", params)
    print(cursor.fetchall()[0][1])
finally:
    if cursor is not None: cursor.close()
    if conn is not None: conn.close()
```

`INSERT INTO`와 `SELECT` 쿼리를 실행할 때, prepared statement를 사용하고 있습니다. placeholder(`%s`)를 사용하여 변수를 입력받을 수 있는 쿼리 템플릿을 만들어 놓고, `params`를 사용하여 해당 쿼리에서 사용할 변수를 함께 DBMS로 제출합니다.


# prepared statement가 필요한 이유


## 1. SQL injection
첫번째는 보안 때문입니다. statment만 사용하여 DBMS에 쿼리를 제출할 경우 개발자가 의도하지 않은 쿼리가 제출되지 않을 수 있습니다. 이러한 해킹 기법을 SQL injection이라고 합니다. 예를 들어 아래와 같이 string으로만 쿼리를 제출한다고 가정해보겠습니다. 

```python
conn = mysql.connector.connect(database="mysql", user="root")
cursor = conn.cursor()


your_name = input()
cursor.execute(f"SELECT * FROM products WHERE name = {your_name}")
```

일반적인 사용자라면 본인의 이름만 입력하겠지만, 어떤 해커가 아래처럼 쿼리를 작성하여 input으로 제출한다면 DB에 본인이 원하는 데이터를 주입하거나, 심지어는 삭제할 수도 있습니다.

```python
your_name = "'ANYTHING' AND 1=1; DELETE FROM products WHERE 1=1"

# 위 입력을 받는 경우에 실제 실행되는 쿼리는 아래와 같습니다
"""
SELECT * 
FROM products
WHERE name = 'ANYTHING' AND 1=1;
DELETE FROM products WHERE 1=1 
"""
```
위처럼 prepared statement를 사용하지 않으면 의도와 다른 쿼리가 실행되는 것을 막을 수 없습니다. 


## 2. 유지 보수

prepared statement를 사용하면 유지보수가 편해집니다. 어플리케이션에서 사용할 쿼리 템플릿이 정해져있고, 템플릿에 입력할 변수를 관리하기 때문입니다. 예를 들어, 어플리케이션에 쿼리를 제출할 때마다 문자열 기반의 쿼리를 작성하는 것은 언제 어떤 쿼리를 제출했는지 추적하기 어렵습니다. 이는 추후에 쿼리를 수정하는 등의 유지 보수가 필요한 상황이 발생했을 때, 코드를 관리하고 수정하기 어렵게 만듭니다. 이에 반해 prepared statment로 제출할 쿼리를 미리 정의해놓았기 때문에 추후에 쿼리를 수정해야할 일이 있을 때 수정해야할 부분을 명확하게 알 수 있습니다.

## 3. Query Caching

prepared statement의 장점 중 쿼리의 반복 실행 시 효율성이 보장되는 가장 큰 이유이며 prepared statement를 사용하는 가장 큰 이유라고도 할 수 있습니다. 

DB는 일반적인 statment를 입력받으면 아래의 세 과정을 거칩니다.

1. Query parsing
2. Optimization
3. Execution

그러나 prepared statement를 입력받는 경우에 DB는 첫 입력에만 1~3의 과정을 거치고 다음 입력부터는 캐시에 저장해둔 정보를 기반으로 쿼리를 실행합니다. 즉, 반복 횟수가 많아질수록 prepared statment가 더 유리해집니다. 이런 점 때문에 prepared statement가 반복되는 쿼리에 있어서 더 효율적이라고 말합니다.

# Query Caching 이란?

쿼리 캐싱은 과거에 실행된 쿼리와 정확하게 같은 쿼리가 실행되었을 때, 과거에 실행된 쿼리 결과를 캐시에 저장해두었다가 반환해주는 것입니다. 이러한 쿼리 캐싱은 `SELECT` 쿼리에 대해서만 동작합니다. 같은 결과를 조회하는 동일한 쿼리에 대해서 두번째 실행부터는 쿼리 분석, 컴파일, 실행의 과정을 거치지 않고 미리 저장해둔 결과를 반환합니다.  

다만, 리서치를 조금 해보니 최근 버전의 DBMS에서는 해당 기능을 점차 지원하고 있지 않는 것으로 보입니다. MySQL도 8.0버전 이후로는 쿼리 캐시 기능을 제거하였으며 MariaDB도 10.1.7버전 이후로는 기본값이 비활성화인 것으로 보입니다. 이렇게 쿼리 캐싱 기능이 점차 비활성화되고 있는 이유는 멀티 코어 시스템에서는 쿼리 캐싱 기능이 확장되지 않기 때문이라고 합니다.

쿼리 캐싱이 동작하기 위해서는 제한 조건이 존재합니다. 
1. 정확하게 같은 쿼리여야한다: 작성된 쿼리의 대소문자까지도 정확하게 일치해야합니다. 이는 쿼리 캐싱을 찾을 때 해당 쿼리의 byte를 검색 key로 사용하기 때문입니다.
2. 조회하는 테이블에 UPDATE/DELETE/INSERT와 같은 데이터 변화가 없어야한다: 쿼리 캐싱된 결과는 해당 테이블이 업데이트되면 즉시 삭제됩니다. 따라서 빈번한 읽기가 발생하는 테이블에 대해서 쿼리 캐싱은 기대 효과를 얻기 어렵습니다.


[참고]  
[[wikipedia]프리페어드 스테이트먼트](https://ko.wikipedia.org/wiki/%ED%94%84%EB%A6%AC%ED%8E%98%EC%96%B4%EB%93%9C_%EC%8A%A4%ED%85%8C%EC%9D%B4%ED%8A%B8%EB%A8%BC%ED%8A%B8)  
[[youtube]코딩애플-SQL injection 공격](https://youtu.be/FoZ2cucLiDs?si=kele-Cb_tPCcCk-v)  
[MySQL Query Cache은 무조건 좋을까? (Feat. query cache lock)](https://jupiny.com/2021/01/10/mysql-query-cache-disadvantage/)