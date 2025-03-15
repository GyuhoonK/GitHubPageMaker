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

Airflow의 variables에 값을 저장해두면, 실행 중인 airflow의 어디서든 해당 값을 불러와 사용할 수 있습니다.  
Web UI에서는 [Admin]>[Variables]에서 값을 추가/삭제할 수 있고, 대량으로 값을 입력해야하는 경우에는 json 파일을 작성하여 업로드할 수도 있습니다.  
아래와 같이 CLI를 이용하는 방법도 있습니다.

```bash
# test.json을 variables로 업로드
airflow variables import test.json
```

CLI에는 get, set, list 등 다양한 [sub command](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#variables)가 존재합니다

airflow variables는 아래 세 가지 특징으로 설명할 수 있습니다.

### [1] key/value store
> a general key/value store

airflow 공식 문서에서 variables를 위와 같이 설명합니다.  
json 파일을 이용하여 variables를 추가하는 부분에서 예상하셨을 수도 있지만, airflow variables는 key/value 포맷입니다. 따라서 아래와 같이 저장됩니다. 
```markdown
| Key       | Value          |
|-----------|----------------|
| example_1 | value_1        |
| example_2 | value_2        |
| example_3 | value_3        |
```

### [2] Json Deserialize
variables는 json deserialize 사용할 수 있습니다. 따라서 아래와 같이 입력되있는 경우에 
```markdown
| Key       | Value                   |
|-----------|-------------------------|
| bar       | {'a': 'value_1'}        |

```

아래 코드에서 bar는 dict로 저장됩니다.

```python
>>> bar = Variable.get("bar", deserialize_json=True)
>>> print(bar)
{'a': 'value1'}
```

jinja template으로 사용하는 경우에도 마찬가지로 json deserialize를 적용할 수 있습니다.
```bash
# Auto-deserialize JSON value
echo {{ var.json.bar }}
{
  "a": "value_1"
}
```

### [3] 암호화 (fernet_key)
variables는 기본적으로 [fernet](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/fernet.html)을 이용하여 암호화를 수행합니다.

fernet key는 airflow 시작 시에 자동으로 생성되며 `[core]` 섹션의 `fernet_key`에 저장됩니다.

```
[core]
fernet_key = *********YOUR_UNIQUE_FERNET_KEY*********
```

fernet key를 가지고 있지 않으면 variables를 읽거나 조작할 수 없다고 설명합니다.  
variables를 저장하는 DB에서 아래 테이블을 조회해보면 어떤 의미인지 알 수 있습니다. 

```sql
SELECT id, `key`, val, description, is_encrypted
FROM airflow.variable;
```

fernet 암호화가 적용되었기 때문에 `val` 컬럼에 저장된 값(value)이 평문이 아니라 암호화된 값임을 확인할 수 있습니다.  
사실 fernet 암호화는 variables 분 아니라 password, connection을 저장할 때도 적용됩니다.

## Best Practice 

[Best Practices - variables](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#airflow-variables)에서 variables에 대한 사용을 어떻게 가이드하고 있는지 살펴보겠습니다. 

### Bad Examples

```python
# dags/test-dag.py
from airflow.models import Variable

foo_var = Variable.get("foo")  # AVOID THAT

bash_use_variable_bad_1 = BashOperator(
    task_id="bash_use_variable_bad_1", 
    bash_command="echo variable foo=${foo_env}", 
    env={"foo_env": foo_var}  # AVOID THAT
)

bash_use_variable_bad_2 = BashOperator(
    task_id="bash_use_variable_bad_2",
    bash_command=f"echo variable foo=${Variable.get('foo')}"  # AVOID THAT
)

bash_use_variable_bad_3 = BashOperator(
    task_id="bash_use_variable_bad_3",
    bash_command="echo variable foo=${foo_env}",
    env={"foo_env": Variable.get("foo")}  # AVOID THAT
)
```
위 코드에서 3가지의 `AVOID THAT`이 보입니다. 하지만 2가지로 분류할 수 있는데요. 아래와 같이 구분할 수 있습니다.

#### [1] Top level python code에서 variables를 사용함
```python
foo_var = Variable.get("foo")  # AVOID THAT
```
위와 같이 DAG 파일 내에서 variables를 호출하게 되면 dag parsing마다 해당 vairable에 접근해야합니다.  
airflow에서 variable에 접근한다는 것은 MySQL에서 `airflow.variables` 테이블에 쿼리 실행을 요청한다는 의미입니다.   
아래처럼 수정한 뒤 파일을 실행해보면 해당 값에 접근 중임을 알 수 있습니다.  

```python
foo_var = Variable.get("foo")  # AVOID THAT
print(foo_var) # 값 확인을 위해 코드 추가
```

```bash
# 작성된 DAG py 파일을 실행해봅니다
python dags/test-dag.py
foo_var
```


#### [2] Operator에 변수 전달 시 `Variable.get` 사용
```python
bash_use_variable_bad_2 = BashOperator(
    task_id="bash_use_variable_bad_2",
    bash_command=f"echo variable foo=${Variable.get('foo')}"  # AVOID THAT
)
```

위처럼 `BashOperator`에 `Variable.get`을 이용하여 변수를 전달해선 안됩니다. 이 경우에도 [1]과 마찬가지로 dag parsing마다 쿼리 실행 을 요청합니다. 


### [1], [2]와 같은 사용을 피해야하는 이유

airflow 문서에서는 [1], [2]가 Base Examples인 이유를 아래처럼 설명합니다. 

> In top-level code, variables using jinja templates do not produce a request until a task is running, whereas, Variable.get() produces a request every time the dag file is parsed by the scheduler if caching is not enabled. Using Variable.get() without enabling caching will lead to suboptimal performance in the dag file processing. In some cases this can cause the dag file to timeout before it is fully parsed.

scheduler가 dag file을 process할 때마다 variable을 조회하는 요청을 생성하므로, 이는 최적이 아니다(suboptimal)라고 설명합니다. 

dag file processing이란, `min_file_process_interval`(default: 30s) 간격으로 dag file을 읽어들여 DAG object로 전환하는 작업입니다. 만약 top level code에서 variable을 사용하게 되면, processing할 때마다 불필요하게 variable을 조회하게 됩니다. 이는 MySQL connection을 과도하게 생성하고 variable을 조회하는 쿼리도 실행하므로 30초마다 airflow에 부하를 줍니다.

이러한 상황을 피해가 위해서 아래와 같은 방법을 사용할 수 있습니다. 

### Good Examples

#### [1] task 내부에서는 자유롭게 사용

```python
@task
def my_task():
    var = Variable.get("foo")  # This is ok since my_task is called only during task run, not during DAG scan.
    print(var)
```

주석에서 설명하고 있는대로, task 내부에서 variable을 호출하면 task execution에서만 variable을 조회하므로 dag processing에서 부하를 주지 않습니다. 

#### [2] jinja template 사용하여 전달
{% raw %}
```python
bash_use_variable_good = BashOperator(
    task_id="bash_use_variable_good",
    bash_command="echo variable foo=${foo_env}",
    env={"foo_env": "{{ var.value.get('foo') }}"},
)
```
{% endraw %}

jinja template을 사용하는 경우 task execution할 때까지 variable을 조회하지 않으므로 부하를 줄일 수 있습니다.


## Variables Cache

어떤 경우에는 위와 같은 Best Examples를 사용하지 못할 수도 있습니다. 이런 경우에는 variable chache를 고려해보아야합니다. 해당 기능은 아직 실험 단계(experimental feature)이긴 하지만, 사용할 수는 있습니다. 

variables cache 긴으을 사용하게 되면, airflow에서 variables를 조회할 때 MySQL을 확인하는 것이 아니라 airflow 내에 cache된 값을 먼저 조회합니다.

이는 실험 기능으로 아직 많이 사용되는 값은 아니기 때문에 관련된 설정값만 살펴보고 마무리하겠습니다.

- [use_cache](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#use-cache)
- [cache_ttl_seconds](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#cache_ttl_seconds)


[참고]  
[[Airflow] Airflow DAG Serialization - 욱이의 IT 생존일지](https://wookiist.dev/168)  
[The ins and outs of Airflow’s new Secrets Cache - Raphaël Vandon](https://medium.com/apache-airflow/the-ins-and-outs-of-airflows-new-secrets-cache-f7b9ec25ca1e)  
[Airflow DAG Parsing - Spidy Web 블로그](https://spidyweb.tistory.com/536)  
[Dag Serialization](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/dag-serialization.html)  
[Dag File Processing](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dagfile-processing.html#dag-file-processing)