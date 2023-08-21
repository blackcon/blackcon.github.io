---
title: Type Confusion 취약점 이해\: 예시와 방지 방법
categories: [Hacking, Theory]
tags: [Hacking, System hacking]
date: 2023-08-20 23:08:00 +0900
---

Type Confusion 취약점은 프로그램의 변수나 객체의 데이터 타입을 잘못 해석하여 비정상적인 동작을 유발하는 보안 취약점입니다. 이번 글에서는 Type Confusion 취약점의 개념을 자세히 설명하고, 실제 예시를 통해 그 작동 원리와 방지 방법을 알아보겠습니다.

## Type Confusion 취약점이란?

Type Confusion 취약점은 주로 다형성을 지원하는 프로그래밍 언어에서 발생합니다. 변수나 객체의 실제 데이터 타입과 프로그램이 해석하는 데이터 타입이 일치하지 않을 때, 예상치 못한 결과를 초래할 수 있습니다. 이는 메모리 누수, 비정상적인 동작, 시스템의 불안정 등을 유발할 수 있습니다.

## Type Confusion 취약점 예시

가정해보겠습니다. 아래의 JavaScript 코드는 다른 동물 객체들을 생성하고 소리를 출력하는 프로그램입니다.

```javascript
class Animal {
    makeSound() {
        console.log("동물 소리");
    }
}

class Dog extends Animal {
    makeSound() {
        console.log("멍멍");
    }
}

class Cat extends Animal {
    makeSound() {
        console.log("야옹");
    }
}

function playSound(animal) {
    animal.makeSound();
}

let dog = new Dog();
let cat = new Cat();

playSound(dog);
playSound(cat);
```

그런데, 사용자 입력으로부터 객체를 생성하는 부분에 문제가 있다고 가정해보겠습니다.

```javascript
let userInput = getUserInput(); // 사용자 입력에 따라 "Dog" 또는 "Cat"이 반환됨
let animal = new userInput();
playSound(animal);
```

이 경우, 사용자 입력에 따라 `userInput` 변수가 `Dog` 클래스 또는 `Cat` 클래스의 이름을 가질 수 있습니다. 그러나 사용자 입력을 신뢰하지 않고 클래스를 생성하면, 원하지 않는 결과를 초래할 수 있습니다. 예를 들어, 사용자가 `"Cat"` 대신 `"Dog"`를 입력했을 경우, `playSound` 함수가 `Dog` 클래스의 소리 대신 `Cat` 클래스의 소리를 출력할 것입니다. 이는 Type Confusion 취약점의 일종입니다.

## Type Confusion 취약점 방지 방법

Type Confusion 취약점을 방지하기 위해서는 다음과 같은 방법을 고려해야 합니다.

1. **사용자 입력 검증**: 사용자 입력을 신뢰하지 않고, 유효한 클래스명인지 검증하는 절차를 추가합니다.

2. **동적 타입 검사**: 클래스 생성 시, 실제로 해당 클래스에 맞는지 검사하는 로직을 추가하여 타입 혼동을 방지합니다.

3. **객체 생성 팩토리**: 객체를 생성하는 부분을 팩토리 패턴 등으로 추상화하여 사용자 입력에 따라 적절한 객체를 생성합니다.

## 결론

Type Confusion 취약점은 다형성을 활용하는 프로그래밍 언어에서 발생하는 중요한 보안 문제입니다. 프로그램에서 사용자 입력을 신뢰하지 않고, 객체 생성과 관련된 부분을 신중하게 다루는 것이 중요합니다. Type Confusion을 방지하기 위해 검증, 동적 타입 검사, 객체 생성 팩토리 등의 접근 방식을 사용하여 프로그램의 안전성을 확보하고 보안 취약점을 방지할 수 있습니다.