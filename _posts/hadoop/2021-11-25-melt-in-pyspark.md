---
layout: post
current: post
cover: assets/built/images/melt-image.jpg
navigation: True
title: Melt in Pyspark
date: 2021-11-25 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

pyspark로 melt function 구현하기



# Melt

pandas에는 melt을 기본으로 제공하고 있습니다(`pandas.melt`).

> pandas.**melt**(*frame*, *id_vars=None*, *value_vars=None*, *var_name=None*, *value_name='value'*, *col_level=None*, *ignore_index=True*)[[source\]](https://github.com/pandas-dev/pandas/blob/v1.3.4/pandas/core/reshape/melt.py#L43-L163)
>
> Unpivot a DataFrame from wide to long format, optionally leaving identifiers set.
>
> This function is useful to massage a DataFrame into a format where one or more columns are identifier variables (id_vars), while all other columns, considered measured variables (value_vars), are “unpivoted” to the row axis, leaving just two non-identifier columns, ‘variable’ and ‘value’.

이를 unpivot한다고 말합니다. pandas 공식 문서에서는 이에 대한 예시를 아래와 같이 보여주고 있습니다.

```python
>>> df = pd.DataFrame({'A': {0: 'a', 1: 'b', 2: 'c'},
                 	     'B': {0: 1, 1: 3, 2: 5},
                       'C': {0: 2, 1: 4, 2: 6}})
>>> df
   A  B  C
0  a  1  2
1  b  3  4
2  c  5  6

>>> pd.melt(df, id_vars=['A'], value_vars=['B'])
   A variable  value
0  a        B      1
1  b        B      3
2  c        B      5

>>> pd.melt(df, id_vars=['A'], value_vars=['B', 'C'])
   A variable  value
0  a        B      1
1  b        B      3
2  c        B      5
3  a        C      2
4  b        C      4
5  c        C      6
```

pyspark는 `pandas.melt`와 같은 함수를 기본으로 제공하지 않지만, 이와 동등한 기능을 pyspark로 구현할 수 있습니다.



# melt in pySpark

```python
from pyspark.sql.functions import array, col, explode, lit, struct
from pyspark.sql import DataFrame
from typing import Iterable 

def melt(
        df: DataFrame, 
        id_vars: Iterable[str], value_vars: Iterable[str], 
        var_name: str="variable", value_name: str="value") -> DataFrame:
    """Convert :class:`DataFrame` from wide to long format."""

    # Create array<struct<variable: str, value: ...>>
    _vars_and_vals = array(*(
        struct(lit(c).alias(var_name), col(c).alias(value_name)) 
        for c in value_vars))

    # Add to the DataFrame and explode
    _tmp = df.withColumn("_vars_and_vals", explode(_vars_and_vals))

    cols = id_vars + [
            col("_vars_and_vals")[x].alias(x) for x in [var_name, value_name]]
    return _tmp.select(*cols)
```



아래와 같은 DataFrame에 pyspark에서 구현한 `melt`가 적용되는 단계를 분석하며, 어떻게 구현했는지 살펴보겠습니다.

```python
>>> pdf = pd.DataFrame({'A': {0: 'a', 1: 'b', 2: 'c'},
                   		  'B': {0: 1, 1: 3, 2: 5},
                  	    'C': {0: 2, 1: 4, 2: 6}})
>>> sdf = spark.createDataFrame(pdf)
sdf.show()
+---+---+---+
|  A|  B|  C|
+---+---+---+
|  a|  1|  2|
|  b|  3|  4|
|  c|  5|  6|
+---+---+---+

```

melt는 아래와 같이 적용해보겠습니다.

```python
melt(sdf, id_vars=['A'], value_vars=['B', 'C']).show()
```



## Step1 : struct

```python
_vars_and_vals = array(*(
        struct(lit(c).alias(var_name), col(c).alias(value_name)) 
        for c in value_vars))
```

`value_vars`에 해당하는 ['B', 'C'] 컬럼에 대해서 각각 StructType의 칼럼을 추가합니다. 기존의 `sdf` 는 아래와 같은 컬럼이 추가됩니다.

```python
+---+---+---+-------------------+
|  A|  B|  C|    varibale, value|
+---+---+-----------------------+
|  a|  1|  2|  [(B, 1), (C, 2)] |
|  b|  3|  4|  [(B, 3), (C, 4)] |
|  c|  5|  6|  [(B, 5), (C, 6)] |
+---+---+---+-------------------+
```

## Step2 : explode

추가된 컬럼에 `explode`를 적용합니다.

```python
+---+---+---+-------------------+-----------------+
|  A|  B|  C|    varibale, value|   _vars_and_vals|
+---+---+---+-------------------+-----------------+
|  a|  1|  2|  [(B, 1), (C, 2)] |            (B,1)| 
|  a|  1|  2|  [(B, 1), (C, 2)] |            (C,2)|
|  b|  3|  4|  [(B, 3), (C, 4)] |            (B,3)| 
|  b|  3|  4|  [(B, 3), (C, 4)] |            (C,4)|
|  c|  5|  6|  [(B, 5), (C, 6)] |            (B,5)|
|  c|  5|  6|  [(B, 5), (C, 6)] |            (C,6)|
+---+---+---+-------------------+-----------------+
```

## Step3 : select columns

`explode`를 통해 만들어진 컬럼 `_vars_and_vals`에서 각각 `variable, value`를 하나씩 꺼내서 독립적인 column으로 만듭니다.

```python
+---+---+---+-------------------+-----------------+----------+-------+
|  A|  B|  C|    varibale, value|   _vars_and_vals|  varibale|  value|
+---+---+---+-------------------+-----------------+----------+-------+
|  a|  1|  2|  [(B, 1), (C, 2)] |            (B,1)|         B|      1|
|  a|  1|  2|  [(B, 1), (C, 2)] |            (C,2)|         C|      2|
|  b|  3|  4|  [(B, 3), (C, 4)] |            (B,3)|         B|      3|
|  b|  3|  4|  [(B, 3), (C, 4)] |            (C,4)|         C|      4|
|  c|  5|  6|  [(B, 5), (C, 6)] |            (B,5)|         B|      5|
|  c|  5|  6|  [(B, 5), (C, 6)] |            (C,6)|         C|      6|
+---+---+---+-------------------+-----------------+----------+-------+
```

필요한 column(`id_vars, variable, value`)만 선택하여 반환합니다.

```python
+---+----------+-------+
|  A|  varibale|  value|
+---+----------+-------+
|  a|         B|      1|
|  a|         C|      2|
|  b|         B|      3|
|  b|         C|      4|
|  c|         B|      5|
|  c|         C|      6|
+---+----------+-------+
```



`pandas.melt` 와 동일한 결과가 나온 것을 확인할 수 있습니다!

[참고]

[How to melt Spark DataFrame?(stackoverflow)](https://stackoverflow.com/questions/41670103/how-to-melt-spark-dataframe)

