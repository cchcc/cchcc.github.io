---
layout: post
title: Destructuring Map in Kotlin
excerpt: "Map 디스트럭쳐링 해보기"
date: 2017-09-05
author: cchcc
categories: blog
tags: [Kotlin, Destructuring]
image:
  url: https://i.imgur.com/Nh3Idw3.gif
comments: true
share: true
---

Map 이라던가 외부 라이브러리의 클래스등을 반복해서 사용하다보면 (혹은 멤버의 일부만 필요하다고 한다면) 이것들을
쪼개서(destructuring) 받고 싶을때가 있습니다.  
가령 아래와 같은 Map 이 있다고 한다면,
