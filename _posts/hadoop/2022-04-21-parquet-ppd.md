---
layout: post
current: post
cover: assets/built/images/hadoop/parquet-logo.png
navigation: True
title: Parquet and Predicate PushDown
date: 2022-04-21 22:30:00 +0900
tags: [hadoop]
class: post-template
subclass: 'post tag-hadoop'
author: GyuhoonK
---

parquet 포맷과 predicate pushdown에 대해서



# Apache Parquet

[Apache Parquet](https://parquet.apache.org/)은 중첩 데이터를 효율적으로 저장할 수 있는 컬럼 기준 저장 포맷입니다. 컬럼 기준 저장 포맷은 파일 크기와 쿼리 성능 측면에서 모두 효율성이 높습니다. 동일한 컬럼의 값을 나란히 모아서 저장하기 때문에 인코딩 효율이 높기 때문에 row 기반 포맷(ex. csv)에 비해 파일 크기가 작습니다. ㄷ또한 쿼리 실행에 필요하지 않은 컬럼은 처리하지 않기 때문에 쿼리 성능이 높습니다.

## DataType

parquet 포맷의 가장 큰 장점은 중첩 구조의 데이터를 저장할 수 있다는 것입니다. 이는 Dremel의 [논문](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36632.pdf)에서 소개한 기술을 적용한 결과입니다. 결과적으로 parquet은 중첩된 필드를 다른 필드와 상관없이 독립적으로 읽을 수 있으며 이를 통해 성능을 향상시킬 수 있었습니다. 

parquet은 다음와 같은 기본자료형을 갖습니다.

| type                 | description                                       |
| -------------------- | ------------------------------------------------- |
| boolea               | 바이너리 값                                       |
| int32                | 부호 있는 32비트 정수                             |
| int64                | 부호 있는 64비트 정수                             |
| int96                | 부호 있는 96비트 정수                             |
| float                | single precision(32비트) IEEE 754 부동소수점 숫자 |
| double               | double precision(64비트) IEEE 754 부동소수점 숫자 |
| binary               | 순차 8비트 부호 없는 바이트                       |
| fixed_len_byte_array | 고정길이 8비트 부호 없는 바이트                   |

각 필드는 반복자(required, optional, repeated), type, name으로 구성됩니다. 간단한 parquet schema 예시는 다음과 같습니다.

```
message WeatherRecord{
	required int32 year;
	required int32 temperature;
	required binary stationID(UTF8);
}
```

특이하게, 문자열 자료형이 존재하지 않습니다. 위의 `stationID` 는 `binary`에 대한 해석 방법으로 `UTF8` 을 사용할 것을 지정하고 있습니다. 이처럼 parquet은 기본자료형에 대해 해석 방식을 정의하는 논리 자료형을 정의하고 있습니다.

논리 자료형 중에 특히 `LIST`와 `MAP` 은 중첩 스키마를 가능하게 합니다.

```
message m{
	required group a (LIST){
		repeated group list {
			required int32 element;
		}
	}
}

message m{
	required group a (MAP) {
		repeated group key_value {
			required binary key (UTF8);
			optional int32 value;
		}
	}
}
```

 이러한 중첩 구조를 저장할 때 Dremel이 제안한 인코딩 방법을 사용합니다. 스키마의 모든 기본자료형 필드의 값을 별도의 컬럼에 저장하고 그 구조는 명세 수준과 반복 수준의 두 정수로 인코딩합니다. 단층 레코드는 null을 사용하고 중첩이나 반복 수준이 올라가면 null이 아닌 값을 사용해서 비트 필드를 인코딩하는 일반적인 기법으로 명세 수준과 반복 수준을 젖아합니다. 이러한 방법으로 중첩 컬럼을 포함한 어떤 컬럼도 다른 컬럼과 상관없이 읽을 수 있습니다. 특히 parquet은 맵의 어떤 value도 읽지 않고 key만 읽을 수도 있습니다. 

## Structure

<img src="../../assets/built/images/hadoop/parquet-structure.gif" alt="image" />

parquet은 크게 header, block, footer로 구성됩니다.





[참고]

하둡 완벽 가이드
