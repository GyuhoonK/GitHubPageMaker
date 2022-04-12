---
layout: post
current: post
cover: assets/built/images/overflow-banner.jpg
navigation: True
title: EXP-SUM-LOG / LOG-SUM-EXP
date: 2021-12-29 22:30:00 +0900
tags: [database]
use-math: True
class: post-template
subclass: 'post tag-database'
author: GyuhoonK
---

 EXP-SUM-LOG / LOG-SUM-EXP trick에 대해서

# SUM in SQL

SQL은 `SUM` 함수를 이용하여 하나의 컬럼 값을 모두 더할 수 있습니다.

아래와 같은 간단한 테이블에 대해서 예시를 들어보겠습니다.

```sql
SELECT * 
FROM default.t_test;
+---+
|  a|
+---+
|  1|
|  2|
|  3|
|  4|
|  5|
+---+

SELECT SUM(a) 
FROM default.t_test;
+------+
|sum(a)|
+------+
|    15|
+------+
```



# Multiplication in SQL

`SUM`함수는 기본으로 제공되지만, column의 값을 모두 곱하는 기능은 기본으로 제공되지 않습니다. 이는 `EXP-SUM-LOG`를 이용해 구현 가능합니다.

EXP-SUM-LOG로 column곱을 구현할 수 있는 이유는 아래와 같은 log의 성질 때문입니다.


$$
log(xy) = log(x) + log(y)\\
x = exp(log(x))
$$


따라서 아래와 같은 수식이 성립합니다.


$$
\sum_{i=1}^nlog(x_i)=log(x_1x_2...x_n)\\
\therefore exp[\sum_{i=1}^nlog(x_i)] = x_1x_2...x_n
$$


 이를 쿼리로 작성하여 실행하면, 아래와 같은 결과가 출력됩니다.

```sql
SELECT EXP(SUM(LOG(a))) AS multiplication
FROM default.t_test;
+--------------+
|multiplication|
+--------------+
|119.9999999...|
+--------------+
```

1~5를 곱한 값은 120이지만, 계산된 값은 119.9999로 다소 오차가 발생했습니다. 이는 LOG를 거치며 정수가 소수로 변환될 때 부동소수점으로 표현이 되고 이에 따라 소수값 연산에 오차가 발생하기 때문입니다.

따라서 정확한 계산 결과를 반환하려면 아래와 같이 ROUND를 적용해주어야합니다.

```SQL
SELECT ROUND(EXP(SUM(LOG(a))), 1) AS multiplication
FROM default.t_test;
+--------------+
|multiplication|
+--------------+
|           120|
+--------------+
```

# LOG-SUM-EXP trick

반대로, `LOG-SUM-EXP`를 이용한 트릭도 존재합니다. 이는 너무 큰 수를 연산하는 경우에 발생하는 **overflow**, 혹은 너무 작은 수를 연산하는 경우엔 발생하는 **underflow**를 피하기 위한 트릭입니다.

$$exp(100) + exp(200)$$을 계산하는 과정을 수식으로 보면 아래와 같습니다.


$$
exp(100) + exp(200) \\
=\dfrac {exp(100) + exp(200)}{exp(100)} * exp(100)\\
=[exp(0)+exp(100)]*exp(100)\\
=exp(log([1+exp(100)]*exp(100)))\\
=exp(log[1+exp(100)] + log(exp(100)))\\
=exp(log[1+exp(100)] + 100)
$$


간단하게 python으로 이를 확인해보았습니다.

```python
from numpy import log 
from numpy import exp

num1 = 3
num2 = 4
print(exp(num1)+exp(num2))
# 74.68368695633191

exp1 = exp(num1-num1)
exp2 = exp(num2-num1)
print(exp(log(exp1+exp2) + num1))
# 74.68368695633188
```

부동소수점 계산에 따른 오차는 있으나, 같은 값이라 봐도 무방할 값이 나왔습니다. 이러한 방식의 계산트릭은 underflow, overflow가 발생하는 경우에 이를 회피하여 값을 기록하는 트릭으로 이용될 수 있습니다.

```python
num1 = 750
num2 = 751

print(np.exp(num1)+np.exp(num2))
# inf

exp1 = exp(num1-num1)
exp2 = exp(num2-num1)

print(log(exp1+exp2) + num1)
# 751.3132616875182

# exp(750)+exp(751)은 계산된 적 없으나 
# 계산결과가 exp(751.3132616875182)이라는 사실은 알 수 있습니다.
```



[참고]

[Multiplication aggregate operator in SQL](https://www.py4u.net/discuss/823103)

[log-sum-exp trick](https://basicstatistics.tistory.com/entry/logsumexp-trick)

