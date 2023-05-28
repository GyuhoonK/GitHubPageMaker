---
layout: post
current: post
cover: assets/built/images/hadoop/determinstic-vs-non-deterministic.webp
navigation: True
title: non-deterministic UDF
date: 2023-05-16 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

non-determinitic UDF 구현하기

### non-deterministic function이란?

`non-deterministic`이라고 했으니, `deterministic`가 아닌 UDF입니다. 먼저, deterministic function이란 무엇인지 살펴보았습니다.   
[(Deterministic and nondeterministic functions)](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/)

>Deterministic functions always return the same result any time they're called with a specific set of input values and given the same state of the database.

>Nondeterministic functions may return different results each time they're called with a specific set of input values even if the database state that they access remains the same.

deterministic function은 같은 입력(input)에 대해서 항상 같은 결과를 반환하는 함수입니다. 반대로
non-deterministic은 deterministic이 아닌 함수로서, 같은 입력에 대해서 다른 결과를 반환할 수 있는 함수입니다. 

우리가 알고 있는 대부분의 함수는 deterministic function입니다. 

```sql
SELECT AVG(col1) AS avg_col1
FROM my_table
```

`AVG` 함수는 my_table에 저장된 데이터가 변하지 않는 한, 같은 입력(`col1`)에 대해서 항상 같은 값을 반환합니다. 

반대로 아래와 같은함수들은 non-deterministic function입니다.

- `current_timestamp`처럼 현재 시간을 출력하는 함수
- `rank`, `row_number`처럼 같은 컬럼 내에서도 다른 값을 부여하는 함수
- `rand`, `uuid`처럼 랜덤 값을 부여하는 함수

위 함수들은 모두 같은 입력에 대해 여러번 실행했을 때 실행 시마다 다른 결과를 반환할 수 있습니다. 


### stateful function이란? 

>If a UDF stores state based on the sequence of records it has processed, it is stateful. A stateful UDF cannot be used in certain expressions such as case statement and certain optimizations such as AND/OR short circuiting don't apply for such UDFs, as they need to be invoked for each record. row_sequence is an example of stateful UDF. A stateful UDF is considered to be non-deterministic, irrespective of what deterministic() returns.

UDF가 실행되는 과정에서 레코드들에 대한 순서 상태(state based on the sequence)를 저장하고 있는 경우 이를 stateful하다고 말합니다. `row_sequence`가 이에 해당합니다. `row_sequence`는 레코드를 정렬하고 해당 레코드의 순서를 기억하고 있기 때문입니다.
실제로 `row_seqence`를 구현한 코드를 보면 더 직관적으로 이해할 수 있습니다. 

```java
@UDFType(deterministic = false, stateful = true)
public class UDFRowSequence extends UDF
{
  private LongWritable result = new LongWritable();

  public UDFRowSequence() {
    result.set(0);
  }

  public LongWritable evaluate() {
    result.set(result.get() + 1);
    return result;
  }
}
```

함수가 실행되는 동안에 `result`에 저장된 값을 참고하여 1씩 증가해나갑니다.

### non-deterministic UDF 정의하기

위의 `row_sequence`에서 보았던 예시처럼 어노테이션을 추가합니다.   
`deterministic = false, stateful = true`은 non-deterministic & stateful UDF를 정의하겠다는 의미입니다.

```java
@UDFType(deterministic = false, stateful = true)
public class UDFNonDetStatfule extends UDF
{
    public Text evaluate() {
        // 함수 정의
    }
}
```

단순히 non-deterministic UDF를 정의하려면 `stateful = False`로 선언하면 됩니다(기본값이 False이므로 인자를 전달하지 않으면 됩니다).

```java
@UDFType(deterministic = false)
public class UDFNonDet extends UDF
{
    public Text evaluate() {
        // 함수 정의
    }
}
```


### Impala에서는 non-deterministic UDF를 사용할 수 없다

이처럼 Hive에서는 non-deterministic, stateful UDF를 지원하지만 impala에서는 이를 지원하지 않습니다.

>All Impala UDFs must be deterministic, that is, produce the same output each time when passed the same argument values. For example, an Impala UDF must not call functions such as rand() to produce different values for each invocation. It must not retrieve data from external sources, such as from disk or over the network.
An Impala UDF must not spawn other threads or processes.

cloudera 공식 문서에서는 impala UDF의 구현 시 제한 사항에 대해 설명하고 있습니다. `rand()`와 같은 함수를 호출하여 각 invacation(함수의 실행/호출)이 다른 값을 생성해서는 안된다고 말합니다. 즉, non-deterministic UDF는 구현이 불가능하며 impala에서 사용하는 모든 UDF는 반드시 deterministic해야합니다.

물론 impala의 built-in function 중에는 non-deterministic function이 존재합니다(`current_timestamp, row_num, rank, uuid` 등). 특히 `uuid`는 비교적 나중에 추가된 함수인데 이 함수를 추가하는 [커밋](https://github.com/cloudera/Impala/commit/12d605d3c20f431d709a515ab5d34615cba0d9e7#diff-b6f339610307238fc9855de9fd8fde22fd65ed11204a8accb2e6fe11255d6c2f)을 확인해보면 java가 아니라 C/C++을 이용했음을 확인할 수 있습니다.

```cpp
void UtilityFunctions::UuidPrepare(FunctionContext* ctx,
    FunctionContext::FunctionStateScope scope) {
  if (scope == FunctionContext::THREAD_LOCAL) {
    if (ctx->GetFunctionState(FunctionContext::THREAD_LOCAL) == NULL) {
      boost::uuids::random_generator* uuid_gen =
        new boost::uuids::random_generator;
      ctx->SetFunctionState(scope, uuid_gen);
    }
  }
}

StringVal UtilityFunctions::Uuid(FunctionContext* ctx) {
  void* uuid_gen = ctx->GetFunctionState(FunctionContext::THREAD_LOCAL);
  DCHECK(uuid_gen != NULL);
  boost::uuids::uuid uuid_value =
    (*reinterpret_cast<boost::uuids::random_generator*>(uuid_gen))();
  const std::string cxx_string = boost::uuids::to_string(uuid_value);
  return StringVal::CopyFrom(ctx,
      reinterpret_cast<const uint8_t*>(cxx_string.c_str()),
      cxx_string.length());
}

void UtilityFunctions::UuidClose(FunctionContext* ctx,
    FunctionContext::FunctionStateScope scope){
  if (scope == FunctionContext::THREAD_LOCAL) {
    void* uuid_gen = ctx->GetFunctionState(FunctionContext::THREAD_LOCAL);
    DCHECK(uuid_gen != NULL);
    delete uuid_gen;
  }
}

```

### 정리
- non-deterministic function은 같은 input에 대해 다른 값을 반환할 수 있는 함수이다
- stateful function은 함수가 실행되는 동안 그 상태(state)를 유지하는 함수이다.
- Impala에서 non-deterministic UDF는 사용할 수 없다

[참고]  
[Deterministic and nondeterministic functions](https://learn.microsoft.com/en-us/sql/relational-databases/user-defined-functions/deterministic-and-nondeterministic-functions?view=sql-server-ver16)  
[UDFType](https://svn.apache.org/repos/infra/websites/production/hive/content/javadocs/r2.1.1/api/org/apache/hadoop/hive/ql/udf/UDFType.html)  
[Add UUID function](https://issues.cloudera.org/browse/IMPALA-1477?jql=ORDER%20BY%20%22summary%22%20ASC)  
[Limitations and restrictions for Impala UDFs
](https://docs.cloudera.com/runtime/7.2.10/impala-sql-reference/topics/impala-udf-limitations-and-restrictions-for-impala-udfs.html)