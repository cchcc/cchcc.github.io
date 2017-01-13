---
layout: post
title: Kodein을 이용한 Kotlin 의존성 주입
excerpt: "Kotlin의 의존성 주입 라이브러리를 소개합니다."
date: 2017-01-07
modified: 2017-01-13
author: cchcc
categories: blog
tags: [Kotlin, Kodein, Dependency Injection, Dagger]
image:
  url: http://i.imgur.com/p0TzIOM.gif
comments: true
share: true
---

## 의존성 주입
클래스 `Computer`, `CPU` 가 있다고 해봅시다. 클래스 `Computer` 가 `CPU` 를 멤버로 가지고 있으면
이런 관계에서 `Computer`가 `CPU`에 대해 __의존성__ 을 가지고 있다고 말합니다.(`Computer` depends on `CPU`)

```kotlin
class Computer {
  private val cpu = CPU()   // 내부에서 생성
}

class Computer(private val cpu: CPU)  // 외부에서 생성
```

이렇게 `CPU` 를 멤버로 가지고 있을때 이 멤버를 `Computer` 클래스 내부에서 생성하는 것이 아니라 외부에서
생성해서 멤버로 세팅하는 방법을 __의존성 주입__ 이라고 합니다. `CPU` 를 생성해서 멤버로 세팅 되는 시점에
따라 `Computer` 가 생성될 때 되는지 혹은 `Computer` 의 다른 멤버 함수를 호출할 때 되는지에 따라
 *생성자 주입* 또는 *메소드 주입* 등으로 구분을 하기도 합니다.

의존성 주입의 구현은 [전략(Strategy) 패턴](https://en.wikipedia.org/wiki/Strategy_pattern)
을 이용하며 그 장점을 그대로 가져옵니다.

이 짓거리를 함으로써 얻는 기본적인 장점은 클래스간 관계를 느슨하게 만들어 준다는 것입니다. 비슷하지만 다른
동작을 하는 멤버를 런타임에서 결정할 수 있도록 해주고, 설계과정에서 미리 멤버의 인터페이스만 정해두고 실제
구현은 다른 팀원이 하도록 분업을 한다던가, 혹은 내 라이브러리를 이용할 3자가 구현하도록 설계할 수 있습니다.
또 의존성을 스텁이나 mock 으로 만들어서 대상 단위테스트에 정말 유용하게 이용할 수 있습니다.

음... 이게 주 내용은 아닌데 간단하게 설명할래도 도저히 간단하게 설명이 안되네요.

## Kodein
[Kodein](https://github.com/SalomonBrys/Kodein) 은 Kotlin 에서 의존성 주입을 아주
나이스하게 하도록 도와주는 라이브러리 입니다. Kotlin 의 DSL 정의 기능을 이용해서 만들어 졌고 좀 해보니
배우기도 쉽고 사용하기도 쉽습니다. Android 에서도 잘돌아가고 따로 특화된 Android 용 모듈도 있습니다.

썰은 그만하고 실제 코드를 봅시다.

```kotlin
interface CPU
class DualCoreCPU : CPU

interface RAM
class DDR4RAM : RAM

val kodein = Kodein { // 의존성과 그 바인딩 방법 선언
  bind<CPU>() with singleton { DualCoreCPU() }
  bind<RAM>() with singleton { DDR4RAM() }
}

class Computer(val dependencies: Kodein) {
  val cpu: CPU = dependencies.instance()  // 바인딩
  val ram: RAM = dependencies.instance()
}

val computer = Computer(kodein) // 의존성 주입
```

간단합니다. *kodein* 은 의존성들을 담는 일종의 컨테이너 입니다. `Kodein` 블럭 안에 *bind* 로 시작하는
것들이 의존성들이고 이것들을 어떤 방법으로 생성해서 바인딩 시켜줄지를 선언해줍니다. 그 이후에 *computer*
객체를 생성하면서 주입시켜 주었습니다. 그러면 `Computer` 클래스 내부에서 타입 추론을 통해 적절한 의존성을
찾아 바인딩됩니다. 바인딩한다는 말이 좀 생소하다구요? 쉽게 말하자면 멤버에 대입시켜준다는 말입니다.

Kodein 은 여러가지 바인딩하는 방법을 제공해주는데요, 한번 살펴보겠습니다.

## 바인딩 방법들
어떤 바인딩 방법을 선언하느냐에 따라 그것을 받아오는 부분의 코드도 약간씩 달라집니다.

##### Singleton binding
최초 바인딩에서 생성된 객체가 계속 이용됩니다. 의존성을 받는 객체의 생명주기에서 단 하나의 객체만 사용할
경우 유용합니다.

```kotlin
val kodein = Kodein {
  bind<CPU>() with singleton { DualCoreCPU() }
}

class Computer {
  val cpu: CPU = kodein.instance()  // 여러번 바인딩해도 같은 객체가 나옴
}
```

##### Provider binding
호출 할때마다 매번 새로운 객체를 생성하는 함수를 생성합니다.

```kotlin
val kodein = Kodein {
  bind<RAM>() with singleton { DDR4RAM() }
}

class Computer {
  val provideRAM: () -> RAM = kodein.provider()  // 이후 provideRAM() 을 호출
}
```

##### Factory binding
인자 하나를 받아 객체를 생성하는 함수를 생성합니다. 특정 조건에 따라 다른 객체를 만드는 팩토리 패턴을 이용할 때
유용합니다.

```kotlin
val kodein = Kodein {
  bind<RAM>() with factory { type: Int -> when(type) {
    3 -> DDR3RAM()
    4 -> DDR4RAM()
    else -> RAM()
  }
}

class Computer {
  val createRAM: (Int) -> RAM = kodein.factory()  // 이후 createRAM(3) 이런식으로 호출

  val ram: RAM = kodein.with(3).instance()  // Singleton binding 처럼 사용 가능
  val provideRAM: () -> RAM = kodein.with(4).provide()  // Provider binding 처럼 사용 가능
}
```

##### Tagged bindings
만약 같은 타입에 대해 여러가지 바인딩을 한다고 하면 이를 구분해 주기위해 *bind* 함수의 파라매터로 태그를
넣어서 사용할 수 있습니다.

```kotlin
val kodein = Kodein {
  bind<RAM>() with singleton { RAM() }
  bind<RAM>("DDR4") with singleton { DDR4RAM() }
  bind<RAM>("DDR3") with singleton { DDR3RAM() }
}

class Computer {
  val ram0: RAM = kodein.instance()
  val ram1: RAM = kodein.instance("DDR3")
  val ram2: RAM = kodein.instance("DDR4")
}
```

##### Constant binding
단순 상수를 바인딩할때 사용합니다. 설정값 같은것들을 지정할때 유용합니다.

```kotlin
val kodein = Kodein {
  constant("NAME") with "my super computer"
}

class Computer {
  val name: String = kodein.instance("NAME")
}
```

##### 의존성의 의존성 제공하기
의존성이 또 다른 의존성을 가지고 있을 경우 의존성을 받아올때의 방법과 동일하게 선언하면 됩니다.

```kotlin
class GPU
class GraphicCard(val gpu: GPU)

val kodein = Kodein {
  bind<GPU>() with singleton { GPU() }  // 이줄을 나중에 선언해도 잘 동작함
  bind<GraphicCard>() with singleton { GraphicCard(instance()) }
}

class Computer {
  val graphicCard: GraphicCard = kodein.instance()
}
```

이 외에도 스레드당 하나의 객체만 바인딩 하는 **Thread singleton binding** , 바인딩 시점이 아닌
선언 시점에 객체를 생성하는 **Eager singleton binding** 등 다양한 종류가 있습니다.

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
사용하기도 쉽고 좀더 직관적입니다. 다만 필요한 의존성 충분하지 않을 경우 Dagger 는 이를 컴파일 타임에
잡아주지만 Kodein 은 런타임에 그냥 에러를 뱉어버립니다. 그런데 이게 간단한 단위테스트 한번만 해봐도
체크할수 있는 부분이라 큰 단점은 아닌듯합니다. 결론은 Kotlin 에서 의존성 주입 라이브러리를 사용한다면
Dagger 보다는 Kodein 을 추천합니다.

<figure>
	<img src="http://i.imgur.com/9f9mbZK.png" alt="image">
	<figcaption>필요한 의존성이 충분하지 않은경우 런타임에서 마주치는 에러...</figcaption>
</figure>

---

참고 문서 : [https://salomonbrys.github.io/Kodein/](https://salomonbrys.github.io/Kodein/)
