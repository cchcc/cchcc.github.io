---
layout: post
title: Kotlin 디폴트가 final인 이유
excerpt: "왜 Kotlin 에서는 클래스와 멤버의 디폴트가 final 일까요?"
date: 2017-01-22
author: cchcc
categories: blog
tags: [Kotlin, Final, Effective Java]
image:
  url: http://i.imgur.com/bn01DZB.gif
comments: true
share: true
---

발단은 이렇습니다. Android 앱 개발중에 Kotlin + espresso + mockito 로 단위테스트를 작성하려고
했더니 클래스의 디폴트 설정이 final 인것 때문에 mock 이 되지않아 영 번거로운 겁니다. 해결책을 이래저래
찾다보니 '왜 Kotlin 은 디폴트가 final 인거지?' 라는 생각에 한번 파보았습니다.

일단 Kotlin [공식문서](https://kotlinlang.org/docs/reference/classes.html#inheritance)
를 보니 이런 내용이 있습니다.

> By default, all classes in Kotlin are final, which corresponds to Effective Java,
Item 17: Design and document for inheritance or else prohibit it.

이펙티브 자바 17... ?

## 상속에 대한 제대로 된 설계와 문서화를 하지 않을거면 상속을 허용하지 마라

이펙티브 자바 17이 뭔가해서 찾아보니 내용은 대략 이렇습니다.

- 오버라이딩이 가능한 메소드는 반드시 문서화를 해야 한다. 상황에 따라 다르게 구현할수도 있음을 포함한
자세한 내용을 알려줘야 한다.

- 오버라이딩이 가능한 메소드는 하위 클래스를 만들어서 반드시 테스트를 해봐야 한다.

- 생성자에서는 오버라이딩이 가능한 메소드를 호출하면 안된다. 초기화가 아직 안된 필드에 접근해서 동작을 수행하면
원치않는 결과가 나올수 있다.

- *Cloneable* 이나 *Serializable* 을 구현하는 클래스는 생성자때와 같은 이유로 clone 과 readObject
에서 오버라이딩 가능한 메소드를 호출하면 안된다.

- 상속을 이용할때는 위에 것들을 잘 지켜 설계와 문서화를 잘하던가 아니면 이를 허용하지 마라.

상속이란걸 잘 쓴다는게 참 어렵네요. 보다보니 저런것들을 다 신경 써가면서 개발하느니 정말 필요한 부분에서만
상속을 이용하고 그냥 final 쓰는게 낫겠다는 생각이 듭니다.

Kotlin 을 사용하는 다른 개발자들은 어떻게 생각하는지 궁금 해졌습니다.

## open VS final

[Kotlin discuss](https://discuss.kotlinlang.org/) 에 가보니 저와 비슷한 생각을 가지고
open 이냐 final 이냐 하는 [토론글](https://discuss.kotlinlang.org/t/classes-final-by-default/166/2)이 있더군요.
좀더 실무적인 입장에서 많은 의견들이 오갔는데 보면서 각각의 편을 지지하는 이유 몇가지를 추려봤습니다.

##### open 파

- Spring 을 쓰고 있는데 일일이 open 으로 세팅해주기 짜증난다.

- 어떤 라이브러리를 이용하는데 재사용 하려는 클래스가 final 로 되있어서 불편했다.

- 이펙티브 자바 '상속에 ~ 허용하지마라' 는 글로보면 훌륭하지만 실무에서는 최악이다.


##### final 파

- 유지보수 하는데 final 로 해두면 다음 설계때 신경을 안써도 되서 편하다.

- 상속을 오용/남용하는 경우를 너무 많이 봤다. 오버라이딩을 이용해서 해결해야되는 문제상황은 매우 극소수이다.

- 커스터마이징이 필요한 지점은 람다로 받아 처리하도록 구조를 잡으면 상속을 이용할 필요가 없다.

<figure>
	<img src="http://i.imgur.com/OtSxwJa.png" alt="image">
	<figcaption>final 파가 약간 더 우세</figcaption>
</figure>

final 이 불편한 경우는 대체로 자바환경에서 사용되는 라이브러리나 프레임워크를 이용할때이고 일반적인 언어
사용 관점에서는 final 을 기본으로 가는게 맞는것 같아 보입니다. 하지만 Kotlin 을 이용하는 대부분의
환경이 자바환경에서 자바를 대체하는 언어 용도로 사용하고 있기때문에 무시할수도 없으니...


## All-open compiler plugin

그래서 나온게 1.0.6 에 추가된 [All-open compiler plugin](https://blog.jetbrains.com/kotlin/2016/12/kotlin-1-0-6-is-here/)
입니다. open VS final 토론글의 시작도 이를 위한 컴파일러 옵션을 제공해달라는 거였는데 이게 채택이 된겁니다.

이렇게 세팅을 해주고

```groovy
buildscript {
    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-allopen:$kotlin_version"
    }
}

apply plugin: "kotlin-allopen"

allOpen {
    annotation("com.your.Annotation")
}
```

이렇게 사용하면 됩니다.

```kotlin
@com.your.Annotation
annotation class MyFrameworkAnnotation

@MyFrameworkAnnotation
class MyClass // will be all-open
```

Spring 이나 JavaBeans 등을 이용하는 개발자들에겐 환영받을 기능입니다.

## 맺으며...

제가 알고 있는 final 의 장점은 런타임때 다형성을 체크하지 않아 성능상의 약간의 이점이 있다는 정도만 알고
있었습니다. open VS final 토론글을 보면서 언어 설계의 관점이라던가 구조라던가 여러가지를 배울수 있었는데요,
글 처음에 겪었던 상황은 해당 클래스를 open 으로 해버릴까 고민했지만 그냥 mockito 를 이용하지 않고 다른방법으로 해결했습니다.
별거아닌 간단한 단위테스트 하나 작성하려다가 이래저래 파다보니 많은 공부가 됬습니다.
