---
layout: post
current: post
cover: assets/built/images/scala-banner-post.png
navigation: True
title: Learning Apache Spark 3 with Scala (Scala Basic)
date: 2022-01-03 22:30:00 +0900
tags: [scala]
class: post-template
subclass: 'post tag-scala'
author: GyuhoonK
---

Learning Apache Spark 3 with Scala (Scala Basic)

# `var` / `val`

| 구분  | 특징                           |
| ----- | ------------------------------ |
| `val` | VALUES are immutable constants |
| `var` | VARIABLES are mutable          |

```scala
// VALUES are immutable constants.
val hello: String = "Hola!"

// VARIABLES are mutable
var helloThere: String = hello
helloThere = hello + " There!"
println(helloThere)
// Hola! There!

val immutableHelloThere = hello + " There"
println(immutableHelloThere)
// Hola! There
```

# String

f-string, zero padding, concatenate

```scala
val pi: Double = 3.14159265
val numberOne: Int = 1
val truth: Boolean = true
val letterA: Char = 'a'

// f-string
println(f"Pi is about $piSinglePrecision%.3f")
// Pi is about 3.142
// f-string & zero-padding
println(f"Zero padding on the left: $numberOne%05d")
// Zero padding on the left: 00001

// concatenate
println(s"I can use the s prefix to use variables like $numberOne $truth $letterA")
// I can use the s prefix to use variables like 1 true a

// get Result
println(s"The s prefix isn't limited to variables; I can include any expression. Like ${1+2}")
// The s prefix isn't limited to variables; I can include any expression. Like 3

```

# Regex

```scala
val theUltimateAnswer: String = "To life, the universe, and everything is 42."
val pattern = """.* ([\d]+).*""".r // regex pattern
val pattern(answerString) = theUltimateAnswer // answerString gets the answer of regex pattern
val answer = answerString.toInt
println(answer)
// 42
```

# Booleans

```scala
// Booleans
val isGreater = 1 > 2
val isLesser = 1 < 2
val impossible = isGreater & isLesser
val anotherWay = isGreater || isLesser

val picard: String = "Picard"
val bestCaptain: String = "Picard"
val isBest: Boolean = picard == bestCaptain
// true
```

# if - else

```scala
// If / else:
if (1 > 3) println("Impossible!") else println("The world makes sense.")
// {}을 이용하여 여러 줄에 작성 가능
if (1 > 3) {
  println("Impossible!")
  println("Really?")
} else {
  println("The world makes sense.")
  println("still.")
}
```

# Matching

```scala
val number = 2
// number에 해당하는 경우(case)를 실행
number match {
  case 1 => println("One")
  case 2 => println("Two")
  case 3 => println("Three")
  case _ => println("Something else") // others
}
// Two
```

# Loop

```scala
for (x <- 1 to 4) {
  val squared = x * x
  println(squared)
}
// 1 4 9 16
var x = 10
while (x >= 0) {
  println(x)
  x -= 1
}
// 10 9 8 ... 0
x = 0
do { println(x); x+=1 } while (x <= 10) // 일단 실행
// 0 1 2 ... 10
// Expressions
{val x = 10; x + 20}
// 하나의 함수로 취급되어 {} 안의 결과를 return
println({val x = 10; x + 20})
// 30
```

# Functions

```scala
// Functions

// format def <function name>(parameter name: type...) : return type = { }

def squareIt(x: Int) : Int = {
  x * x
}

def cubeIt(x : Int) : Int = {x * x * x}

println(squareIt(2))
// 8
println(cubeIt(3))
// 27

// scala에서 함수는 함수를 인자로 받을 수 있다
def transformInt(x: Int, f: Int => Int): Int = {
  f(x)
}

val result = transformInt(2, cubeIt)
println(result)
// 8
transformInt(3, x => x * x * x)
// 27
transformInt(10, x => x / 2)
// 5
transformInt(2, x => {val y = x * 2; y * y})
// 16 (4 * 4)
```



## Data structures

## Tuples : Immutable Lists

```scala
val captainStuff = ("Picard", "Enterprise-D", "NCC-1701-D")
println(captainStuff)

// Refer to the individual fields with a ONE-BASED index
println(captainStuff._1) // Picard 
println(captainStuff._2) // Enterprise-D
println(captainStuff._3) // NCC-1701-D

val picardsShip = "Picard" -> "Enterprise-D"
println(picardsShip._2)
// Enterprise-D

// could be of different type
val aBunchOfStuff = ("Kirk", 1964, true)
```

## Lists : more functionality and must be of same type

```scala
val shipList = List("Enterprise", "Defiant", "Voyager", "Deep Space Nine")

println(shipList(1)) // Defiant
// zero-based

println(shipList.head) 
// Enterprise
println(shipList.tail) // excluding head
/// Defiant, Voyager, Deep Space Nine

for (ship <- shipList) {println(ship)}
// Enterprise Defiant Voyager Deep Space Nine
```

## reduce & filter

```scala
val numberList = List(1, 2, 3, 4,5 )
val sum = numberList.reduce( (x: Int, y: Int) => x + y)
println(sum) // 15

// filter() removes stuff
val iHateFives = numberList.filter( (x: Int) => x != 5)
// List(1, 2, 3, 4)
val iHateThrees = numberList.filter(_ != 3)
// List(1, 2, 4, 5)
```

## concatenate lists 

```scala
val numberList = List(1, 2, 3, 4,5 )
val moreNumbers = List(6,7,8)
val lotsOfNumbers = numberList ++ moreNumbers
// List(1, 2, 3, 4, 5, 6, 7, 8)
val reversed = numberList.reverse
// List(5, 4, 3, 2, 1)
val sorted = reversed.sorted
// List(1, 2, 3, 4, 5)
val lotsOfDuplicates = numberList ++ numberList
// List(1, 2, 3, 4, 5, 1, 2, 3, 4, 5)
val distinctValues = lotsOfDuplicates.distinct
// List(1, 2, 3, 4, 5)
val maxValue = numberList.max
// 5
val total = numberList.sum
// 15
val hasThree = iHateThrees.contains(3)
// true
```

## maps

```scala
al shipMap = Map("Kirk" -> "Enterprise", "Picard" -> "Enterprise-D", "Sisko" -> "Deep Space Nine", "Janeway" -> "Voyager")
println(shipMap("Janeway")) // Voyager
println(shipMap.contains("Archer")) // false
val archersShip = util.Try(shipMap("Archer")) getOrElse "Unknown"
println(archersShip) // Unknown
```

[참고]

[Learning Apache Spark 3 with Scala](https://www.udemy.com/course/best-scala-apache-spark/)
