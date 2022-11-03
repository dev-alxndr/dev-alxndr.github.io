---
title: Java Lambda 외부 변수 사용시 문제
tags: [Java, lambda]
categories: [TIL]
date : 2022-11-03 00:00:00
---

`TransactionTemplate.executeWithoutResult(() -> {....})`

람다를 사용 중 메소드 내부 람다 외부 변수의 값을 변경하려는 도중 컴파일 오류가 발생했고

```
Variable used in lambda expression should be final or effectively final
```

intellij가 AmoticReference로 한번 감싸도록 변경해주는 것을 보고 정리

## TL;DR

-   람다 실행 되던 메소드의 스택영역에 저장되는 변수는 람다 내부에서 참조만 가능하고 값 변경은 불가
-   외부 Reference Type 변수에 대한 변경은 Heap 메모리 데이터를 변경하는 것이기 때문에 가능 (AtomicReference로 감싸준 이유)
-   람다가 실행될 때 캡처링이 일어나면서 발생하는 현상
    -   캡처링이 일어나게 되면
        -   람다의 새로운 스택을 생성
        -   실행되던 메소드의 스택 데이터를 그대로 가져와서 람다의 스택에 복사 (call by value)

---
## Reference
[람다 캡처링 :: Variable used in lambda expression should be final or effectively final의 이유](https://cobbybb.tistory.com/19)