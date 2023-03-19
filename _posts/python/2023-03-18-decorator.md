---
layout: post
current: post
cover: assets/built/images/python/decorator-comic-1.png
navigation: True
title: 변수를 전달하는 decorator
date: 2023-03-18 22:30:00 +0900
tags: [python]
class: post-template
subclass: 'post tag-python'
author: GyuhoonK
---

변수를 생성해 함수로 전달하는 데코레이터 만들기


## decorator를 사용하고 싶었던 이유
요즘 작업 중인 업무 중에, `connection`을 선언한 뒤 해당 커넥션 객체를 이용하여 서버와 통신하는 작업을 많이 하고 있습니다. 예를 들어, DB와 연결하는 connection를 함수내에서 선언하고 이를 이용해서 작업하는 식입니다.

```python
from pyhive import hive

def func():
    conn = hive.Connection(host='hive-host', 
                           port=10000, 
                           username='username', 
                           password='password', 
                           database='default')
    # 무엇인가 작업을 작성함 #
```

그런데 위와 같은 함수를 꽤나 많이 작성해야했습니다. 그러면 매번 함수를 작성할 때마다 함수 내에서 `conn`을 선언해야했습니다. 

```python
def func1():
    conn = hive.Connection(host='hive-host', 
                           port=10000, 
                           username='username', 
                           password='password', 
                           database='default')
    # 무엇인가 작업을 작성함 #

def func2():
    conn = hive.Connection(host='hive-host', 
                           port=10000, 
                           username='username', 
                           password='password', 
                           database='default')
    # 무엇인가 작업을 작성함 #

# 새로운 함수를 작성할 때마다 동일한 코드(conn을 선언)를 작성해야했다
```

클래스를 선언하고 클래스 커넥션 객체를 인스턴스 변수로 선언해놓은 뒤 위에서 선언한 함수를 메소드로 만들면 `conn`을 여러번 선언해야하는 문제가 해결될 수 있긴합니다.

```python
Class HiveJob():
    def __init__(self):
        self.conn = hive.Connection(host='hive-host', 
                               port=10000, 
                               username='username', 
                               password='password', 
                               database='default')
    def func1():
        query = "SELECT * FROM TABLE1"
        return pd.read_sql(query, self.conn)
    
    def func2():
        cursor = self.conn.cursor()
        cursor.execute("DROP TABLE TABLE1")
```

이것이 해결책이 될 수도 있지만 제가 원하는 방향과는 맞지 않았습니다. 첫번째 이유는 `func1, func2`와 같은 함수가 다른 코드에서는 사용되지 않고 전체 코드 실행 중에 거의 1회 정도만 실행되기 때문에 클래스에 적합하지 않는다고 생각했기 때문이고, 두번째는 이렇게 하나의 클래스로 모으기에는 이미 함수가 너무나도 많이 작성되어있는 상태였기때문입니다. 그렇다고 지금까지 작성된 함수는 그대로 두고, 앞으로 작성하려는 함수만 클래스에 모아서 메서드로 관리하게되면 이전에 작성된 함수는 코드 내에 함수로 남고, 앞으로 작성되는 함수만 클래스 내 메서드로 관리되는 이상한 상황이 될 것입니다.

## decorator로 반복되는 코드를 없애자
위와 같은 이유로 (미리 이러한 상황을 예상하지 못한 탓에), 대체할 방법으로 앞으로도 함수로 작성하되 중복되는 코드를 줄일 방법을 고민했습니다. decorator를 사용하면 문제를 해결할 수 있을 것 같았습니다. decorator가 python에 도입된 이유는 함수를 정의한 이후에 필요한 기능을 추가하지 않고, 정의와 동시에 필요한 기능을 추가하기 위해서입니다.  
PEP-318에서는 아래와 같은 예시로 데코레이터의 편리성을 설명합니다. 

```python
def foo(cls):
    pass
foo = synchronized(lock)(foo)
foo = classmethod(foo)
### foo 함수를 선언한 뒤, 필요한 기능(sync, class) 추가를 위해 2줄이 더 붙습니다
### foo 함수 정의를 위해 foo를 3번이나 더 적어야합니다

@classmethod
@synchronized(lock)
def foo(cls):
    pass
### 데코레이터를 이용하면 함수 선언만으로 기능 추가도 완료됩니다
```

제 상황에 대입하여 생각해보니 추가하고자 하는 기능은 `conn` 객체를 만드는 것이고 함수 내부는 커넥션을 이용한 동작만 필요했습니다. 따라서 데코레이터를 사용한다면 아래와 같은 포맷을 생각했습니다. 

```python
@get_connection
def func1(): 
    query = "SELECT * FROM TABLE1"
    return pd.read_sql(query, conn)

@get_connection
def func2():
    cursor = conn.cursor()
    cursor.execute("DROP TABLE TABLE1")
```

`@get_connection`만 있으면 함수 내부에서 `conn`이라는 이름의 커넥션 객체가 생성되고 이를 이용하여 동작하는 부분의 코드만 작성하고 싶었습니다! 위와 같은 사용 사례를 아직 본 적은 없었지만 왠지 가능할 것 같았습니다.

`get_connection` 데코레이터를 작성해보았습니다. 

```python
def get_connection(func): 
    def wrapper(*args, **kwargs):
        conn = hive.Connection(host='hive-host', 
                               port=10000, 
                               username='username', 
                               password='password', 
                               database='default')
        result = func(conn, *args, **kwargs)
        return result
    return wrapper

@get_connection
def func1(conn): 
    query = "SELECT * FROM TABLE1"
    return pd.read_sql(query, conn)

@get_connection
def func2(conn):
    cursor = conn.cursor()
    cursor.execute("DROP TABLE TABLE1")
```

`func1, func2`는 함수 내부에서 커넥션 객체(`conn`)를 생성하지는 않지만, 데코레이터 `get_connection`의 wrapper에서 생성한 `conn`을 전달받습니다. `conn`을 전달받기 위해 함수에 매개변수를 추가했습니다. 데코레이터 문법은 사실 아래처럼 함수를 재정의하는 것과 다름 없습니다. 

```python
func1 = get_connection(func1)
# get_connection 내부의 wrapper의 리턴이 최종적으로 리턴된다
# 따라서 return result 가 없으면, None을 리턴한다
```

따라서, 함수(`func`)의 리턴(`result`)을 리턴해주지 않으면 값을 데코레이터 수식을 받은 함수의 리턴이 `None`이 됩니다.

wrapper는 `*args, **kwargs`로 데코레이터 수식을 받는 함수의 변수를 전달받아 다시 func에 전달하므로, 아래와 같이 변수를 입력해야하는 함수에도 데코레이터를 사용할 수 있습니다.

```python
@get_connection 
def drop_table(conn, table):
    cursor = conn.cursor()
    cursor.execute(f"DROP TABLE {table}")
```

### 아쉬운 점
함수 선언 시에 `conn`을 매개변수로 반드시 추가해야한다는 점이 아쉽습니다. 데코레이터를 붙이기만 하면, 함수 내부에 어떤 변수(객체)가 생성되도록 하는 기능은 불가능한걸까요? 위에서 예시로 들었던 클래스로 구현하는 경우에는 메소드가 인스턴스 변수(`self.conn`)에 직접 접근할 수 있으므로 매개변수로 `conn`을 받지 않았습니다.

```python
## 메소드는 self.conn에 직접 접근 가능하므로 매개변수가 필요 없다
    def func1():
        cursor = conn.cursor()

## 데코레이터에서 생성한 conn을 전달받기 위해 매개변수가 필요하다
def func1(conn): 
    cursor = conn.cursor()
```

`conn` 하나 넣는게 뭐가 불편하다고, 라고 생각하실 수도 있지만 지금까지 작성된 함수들에 `conn` 객체 선언을 삭제하고, 데코레이터를 추가하려고 했더니 매개변수를 추가해야한다는 건 꽤나 큰 부담이었습니다(그래서 추가하지 못했습니다..)  
뭔가 스마트하게 추가할 수 있는 방법이 있다면 좋을텐데 일단은 현재로서도 제가 원했던 기능은 구현이 된 것 같아 추가 조사는 하지 않았습니다.

[참고]  
[PEP 318 – Decorators for Functions and Methods](https://peps.python.org/pep-0318/)  
[데코레이터 만들기](https://dojang.io/mod/page/view.php?id=2427)