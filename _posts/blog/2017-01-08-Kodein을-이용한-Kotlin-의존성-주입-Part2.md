---
layout: post
title: Kodein을 이용한 Kotlin 의존성 주입 - Part 2
excerpt: "Kotlin의 의존성 주입 라이브러리를 소개합니다."
date: 2017-01-08
author: cchcc
categories: blog
tags: [Kotlin, Kodein, Dependency Injection, Dagger]
image:
  url: http://i.imgur.com/p0TzIOM.gif
comments: true
share: true
---

## 의존성 재사용하기

##### Modules
`Kodein` 으로 의존성들을 만들때 자주 이용되는 것들은 모아서 재사용하도록 만들수 있습니다.

```kotlin
val dualCoreModule = Kodein.Module {
  bind<CPU>() with singleton { DualCoreCPU() }
}

val kodein = Kodein {
  import(dualCoreModule)
}
```

##### Extension
기존의 정의한 `Kodein` 객체를 그대로 이용하고 싶을때 사용합니다. 의존성들을 조합하는 일종의 컴포지션
방법입니다.

```kotlin
val cpuKodein = Kodein {
  bind<CPU>() with singleton { DualCoreCPU() }
}

val kodein = Kodein {
  extend(cpuKodein)
}
```

##### Overriding
`Kodein` 에서 선언하는 의존성은 기본적으로 같은 타입의 바인딩을 중복되게 할수는 없지만, *overrides*
를 설정하면 기존것을 대체하도록 만들수 있습니다. 단위 테스트를 할때 매우 유용합니다.

```kotlin
val computerKodein = Kodein {
  bind<CPU>() with singleton { DualCoreCPU() }
  bind<RAM>() with singleton { DDR4RAM() }
}

val testKodein = Kodein {
  extend(computerKodein)
  bind<CPU>(overrides = true) with singleton { mock<CPU>() }
}
```

## Injector
프레임워크 같은것들을 이용해서 개발 할경우 프레임워크가 직접 주요 객체를 생성하고 이후 코더가 객체의 생명주기에
관여할 수 있도록 별도의 메카니즘을 제공해 주기도 하는데요. 즉 의존성이 객체의 생성때 세팅되는게 아니라 다른
시점에 세팅되야 하는경우 이 방법을 이용합니다.

```kotlin
class SuperComputer : Computer() {
    val injector = KodeinInjector()

    val cpu: CPU by injector.instance()
    val ram: RAM by injector.instance()

    override fun onStart() {
        val kodein = Kodein {
            bind<CPU>() with singleton { DualCoreCPU() }
            bind<RAM>() with singleton { DDR4RAM() }
        }

        injector.inject(kodein) // 이 시점에 멤버가 세팅됨
    }
}
```

## Lazy
Kotlin 에서는 *by lazy* 를 이용해여 멤버의 초기화 시점을 최초 read 될때로 늦출수가 있습니다. 이와
같은 기능을 Kodein 에서도 이용할수 있습니다.

```kotlin
class Computer(val kodein: Kodein) {
    val cpu: CPU by kodein.lazy.instance()    // = 이 아니라 by 를 사용함
    val provideRAM: () -> RAM by kodein.lazy.provider()
}
```

의존성을 생성 시점이 아닌 다른 시점에 세팅할때 *injector* 를 이용하는 방법외에도 *kodein* 자체를 lazy
하게 만들어 해결할 수도 있습니다.

```kotlin
class Computer() {
    val kodein = LazyKodein {
        /* kodein 을 어떻게 가져올지 기술, Kodein 객체를 리턴 */
    }

    val cpu: CPU by kodein.instance()
    val provideRAM: () -> RAM by kodein.provider()
}

// Kodein.lazy 을 이용해서 직접적으로 바인딩 선언도 가능함
val kodein = Kodein.lazy {  // 리턴 타입이 LazyKodein
  bind<CPU>() with singleton { DualCoreCPU() }
  bind<RAM>() with singleton { DDR4RAM() }
}
```

## 좀 더 간결하게 만들어 보자
*kodein* 또는 *injector* 를 이용해 의존성을 받아오는 클래스를 좀더 간결하게 쓸 수 있도록 해주는 몇가지
인터페이스가 있습니다.

```kotlin
class Computer : KodeinAware {
    override val kodein = Kodein {
        bind<CPU>() with singleton { DualCoreCPU() }
        bind<RAM>() with provider { DDR4RAM() }
    }

    val cpu: CPU = instance() // 여기가 좀 더 간결해졌다.
    val provideRAM: () -> RAM = provider()
}

class Computer : KodeinInjected {
    override val injector: KodeinInjector = KodeinInjector()

    val cpu: CPU by instance()
    val provideRAM: () -> RAM by provider()
}

class Computer : LazyKodeinAware {
    override val kodein = Kodein.lazy {
        bind<CPU>() with singleton { DualCoreCPU() }
        bind<RAM>() with provider { DDR4RAM() }
    }

    val cpu: CPU by instance()
    val provideRAM: () -> RAM by provider()
}
```

## Dagger 와 비교
Java 쪽에서 이미 유명한 의존성 주입 라이브러리인 [Dagger](https://google.github.io/dagger/)
역시 Kotlin 에서 아주 잘 돌아갑니다. 두가지를 비교해보자면 Kodein 은 Dagger 에 비해 가볍고 배우기도
사용하기도 쉽고 좀더 직관적입니다. 다만 의존성의 의존성 관계가 있을경우 Dagger 는 이를 컴파일 타임에
잡아주지만 Kodein 은 런타임에 그냥 에러를 뱉어버립니다. 그런데 이게 간단한 단위테스트 한번만 해봐도
체크할수 있는 부분이라 큰 단점은 아닌듯합니다. 결론은 Kotlin 에서 의존성 주입 라이브러리를 사용한다면
Dagger 보다는 Kodein 을 추천합니다.

<figure>
	<img src="http://i.imgur.com/9f9mbZK.png" alt="image">
	<figcaption>필요한 의존성이 충분하지 않은경우 런타임에서 마주치는 에러...</figcaption>
</figure>

---

- [Kodein을 이용한 Kotlin 의존성 주입 - Part 1]({{ site.url }}/blog/Kodein%EC%9D%84-%EC%9D%B4%EC%9A%A9%ED%95%9C-Kotlin-%EC%9D%98%EC%A1%B4%EC%84%B1-%EC%A3%BC%EC%9E%85-Part1/)
- Kodein을 이용한 Kotlin 의존성 주입 - Part 2

참고 문서 : [https://salomonbrys.github.io/Kodein/](https://salomonbrys.github.io/Kodein/)
