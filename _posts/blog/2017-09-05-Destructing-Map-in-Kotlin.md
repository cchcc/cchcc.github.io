---
layout: post
title: Destructuring Map in Kotlin
excerpt: "Map 디스트럭쳐링 해보기"
date: 2017-09-05
author: cchcc
categories: blog
tags: [Kotlin, Destructuring]
image:
  url: http://i.imgur.com/Nh3Idw3.gif
comments: true
share: true
---

Map 이라던가 외부 라이브러리의 클래스등을 반복해서 사용하다보면 (혹은 멤버의 일부만 필요하다고 한다면) 이것들을
쪼개서(destructuring) 받고 싶을때가 있습니다.  
가령 아래와 같은 Map 이 있다고 한다면,

```kotlin
val map = mapOf("name" to "Learn You a Haskell for Great Good!"
                , "writer" to "Miran Lipovača"
                , "pages" to 400
                , "contents" to "Introduction|Starting Out|Types and Typeclasses|Syntax in Functions"
                , "web" to "http://learnyouahaskell.com/")
```

아래처럼 쪼개서 사용하고 싶은 것입니다.(깔끔하니까!)

```kotlin
val (name, writer, pages, contents) = map
```

이번글은 이럴때 어떻게 하는게 잘 쪼개는게 좋은 방법일지 그 고민을 적어볼까 합니다.

#### Extensions 를 이용해서?

Kotlin 에는 [Extensions](https://kotlinlang.org/docs/reference/exceptions.html) 라는 별도의 클래스 구현없이 함수를 추가할수 있는 기능이 있습니다.
요걸 이용해 보자면,  

```kotlin
operator fun Map<String, Any?>.component1(): String = this["name"] as String
operator fun Map<String, Any?>.component2(): String = this.getOrDefault("writer", "") as String
operator fun Map<String, Any?>.component3(): Int = this["pages"] as Int
operator fun Map<String, Any?>.component4(): List<String> = (this["contents"] as String).split("|")
operator fun Map<String, Any?>.component5(): String? = this["web"] as String?

val (name, writer, pages, contents) = map
```

괜찮아 보이는듯 하지만 이렇게 해두면 `Map<String, Any?>` 타입의 destructuring 기능이 글로벌하게 못박아 버리는
것이므로 썩 좋지는 않습니다. 그렇다고 사용하는 범위(scope) 를 한정해 주기위해 아래처럼  

```kotlin
class Logic {
  operator fun Map<String, Any?>.component1(): String = this["name"] as String
  operator fun Map<String, Any?>.component2(): String = this.getOrDefault("writer", "") as String
  operator fun Map<String, Any?>.component3(): Int = this["pages"] as Int
  operator fun Map<String, Any?>.component4(): List<String> = (this["contents"] as String).split("|")
  operator fun Map<String, Any?>.component5(): String? = this["web"] as String?

  fun foo() {
    val (name, writer, pages, contents) = map
    // ...
  }
}
```

이렇게 하면 `Logic` 클래스의 범위에서만 사용 가능 하도록 적용이 되지만, `Logic` 외에 Destructuring 을 사용할 클래스마다
*componentN* 함수들을 추가해 주자니 영 번거롭습니다.  

#### 상속(inherit)을 이용해서?

***OOP*** 를 배웠다면 사실 가장 먼저 떠오르는, 가장 쉬운 방법 일것 같습니다.

```kotlin
class Book(map: Map<String, Any?>) : HashMap<String, Any?>(map) {
    operator fun component1(): String = this["name"] as String
    operator fun component2(): String = this.getOrDefault("writer", "") as String
    operator fun component3(): Int = this["pages"] as Int
    operator fun component4(): List<String> = (this["contents"] as String).split("|")
    operator fun component5(): String? = this["web"] as String?
}

val (name, writer, pages, contents) = Book(map)
```

이 방법의 문제점은 다름아닌 ***상속*** 그 자체입니다. 우리는 이미 상속이 주는 단점에 대해 충분히 많은 글들을 봤고
경험도 해왔을 뿐더러 지금 상황이 상속만이 해결할수 있는 ***다형성(polymorphism)*** 과 관련된 문제도 아닙니다.  
그러므로 이 방법은 그냥 안쓰는걸로...

#### 컴포지션(composition) 으로?

상속이 아니면 컴포지션이죠.

```kotlin
interface MapToBookDestructurer {
    operator fun Map<String, Any?>.component1(): String = this["name"] as String
    operator fun Map<String, Any?>.component2(): String = this.getOrDefault("writer", "") as String
    operator fun Map<String, Any?>.component3(): Int = this["pages"] as Int
    operator fun Map<String, Any?>.component4(): List<String> = (this["contents"] as String).split("|")
    operator fun Map<String, Any?>.component5(): String? = this["web"] as String?
}

class Logic : MapToBookDestructurer {
  fun foo() {
    val (name, writer, pages, contents) = map
    // ...
  }  
}
```

나쁘지 않네요. interface 로 만들었으니 부모와 상관없이 맘대로 합성(composition)해서 사용가능하고
쪼개기(destructuring)의 사용 범위도 `Logic` 클래스만 한정되도록 했습니다.  
그런데... 만약 `Logic` 클래스 내부에서 map 의 다른 멤버로 destructuring 이 필요하다면 어떻게 해야 할까요?
아래처럼 말이죠.

```kotlin
interface MapToBookLinkDestructurer {
    operator fun Map<String, Any?>.component1(): String = this["name"] as String
    operator fun Map<String, Any?>.component2(): String? = this["web"] as String?
}

// Error! it inherits multiple interface methods
class Logic : MapToBookDestructurer, MapToBookLinkDestructurer {
  fun foo1() {
    val (name, writer, pages, contents) = map // MapToBookDestructurer
    // ...
  }

  fun foo2() {
    val (name, web) = map  // MapToBookLinkDestructurer
    // ...
  }  
}
```

아... 이럼 안되죠. *component1* 와 *component2* 가 중복되서 컴파일러가 어찌 할줄을 몰라합니다.  
이 상황에도 써먹을수 있는 더 좋은 방법이 없으려나요?

#### Composition + 1 global extension function

이래저래 고민하다보니 괜찮은 방법이 만들어졌습니다.

```kotlin
interface MapToBookDestructurer {
    val map: Map<String, Any?>
    operator fun component1(): String = map["name"] as String
    operator fun component2(): String = map.getOrDefault("writer", "") as String
    operator fun component3(): Int = map["pages"] as Int
    operator fun component4(): List<String> = (map["contents"] as String).split("|")
    operator fun component5(): String? = map["web"] as String?
}

fun Map<String, Any?>.destructuringToBook(): MapToBookDestructurer = object : MapToBookDestructurer {
    override val map = this@destructuringToBook
}


interface MapToBookLinkDestructurer {
    val map: Map<String, Any?>
    operator fun component1(): String = map["name"] as String
    operator fun component2(): String? = map["web"] as String?
}

fun Map<String, Any?>.destructuringToBookLink(): MapToBookLinkDestructurer = object : MapToBookLinkDestructurer {
    override val map = this@destructuringToBookLink
}
```

각 destructuring 할 interface 에 *map* 을 멤버로 추가하고 글로벌 extension 함수 하나를 만들어서 익명 객체를 리턴하도록 만들었습니다.
이렇게 해두면 사용할때 아래처럼 범위에 관계없이 필요한 destructuring 을 사용할수 있습니다.

```kotlin
class Logic {
    fun foo1() {
        val (name, writer, pages, contents, web) = map.destructuringToBook()
    }

    fun foo2() {
        val (name, web) = map.destructuringToBookLink()
    }
}
```

고민은 여기까지입니다. 이 방법이 제일 좋아보이기는 하는데 익명 객체를 생성한다는게 좀 오버헤드가 있어보이네요.

#### 맺으며...
Kotlin 에는 아직 Tuple 기능이 없습니다. 그나마 *componentN* 를 활용해서 쪼개는 기능만이라도 비슷하게 써보려 하다보니
이런 고민도 해봤습니다. `Map<String, Any?>` 타입을 제네릭으로 바꿔 `<T : out Map<String, Any?>>` 로 써서 더 개선 할
수도 있을것 같고 더 나은 방법이 있을거 같기도 합니다.  뭐든 자유롭게 의견 주셔도 좋습니다~  
