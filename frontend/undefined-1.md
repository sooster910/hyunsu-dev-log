---
description: >-
  어떤 한 블로그에서 리스코프 치환 원칙을 통해 타입 시스템을 설명하는 것을 보았는데 흥미로웠다. 사실 리스코프 원칙이 중요한 것 보다
  타입스크립트를 1년 동안 사용 했음에도 원리를 잘 이해하지 못한 것 같아 다시 이해를 목적으로 한 글이다.
hidden: true
---

# 서브 타입화, 할당성



호환성이라는 단어에 대해 공식문서에서도 언어명세에 정의 되어있지 않은 용어라고 한다.

TypeScript에는 두종류의 호환성이 있는데 subtype과 assignment이다.



{% hint style="info" %}
wiki에 따르면 Subtyping은  프로그래밍 언어 이론에서 S 가 T의 서브타입인 경우, \
서브타이핑 관계 (S<:T ) 는 타입 S는 타입 T로 예상되는 모든 상황에서 안전하게 사용될 수 있다.
{% endhint %}



이런 관계가 supertype/subtype으로 부르기도 하는데, 이 관계를 적용하는 방법엔 nominal typing과 structural typing이 있다. 언어에 따라 다른 방식이 적용되는데 타입스크립트는 structural typing(구조적 타이핑)을 사용한다. <mark style="color:orange;">**구조적 타이핑은 해당 멤버들을 기준으로 타입을 결정하는 방식이다**</mark>.\


일단 기본 타입을 바탕으로 서브타입관계를 정리 해보았다.









Dog클래스는 Pet인터페이스의 모든 멤버를 가지고 있기 때문에, Dog인스턴스를 Pet타입의 변수에 할당할 수 있다.

```
interface Pet {
  name: string;
}

class Dog {
  name: string;
  constructor(name: string) {
    this.name = name;
  }
}

let pet: Pet = new Dog("Lassie"); // OK, because Dog has the same members as Pet

```

구조가 같으니 이를 호환한다. 인터페이스나 클래스의 이름이 다르더라도 구조가 같다면 호환이 되는것도 가능하다&#x20;





### LSP로 보는 타입 시스템

먼저 리스코프치환 원칙을 wiki에서는

> if S is subtype of T then objects of type T in a program may be replaced with objects of type S without altering any of the desirable properties of that program
>
> If S is subtype of T, the subtyping relation (S<:T,S≤T ) means that any term of type S can safely be used in any context where a term of type T is expected.

&#x20;서브타입은 그것의 기반 타입으로 치환 가능해야 한다.

객체지향으로 나타내면,

```typescript
// 전통적인 LSP 예시
class Animal {
    makeSound() { return "Some sound"; }
}

class Dog extends Animal {
    makeSound() { return "Woof!"; }
    bark() { return "Bark!"; }
}

// Dog는 Animal의 자리를 대체할 수 있어야 함
function makeAnimalSound(animal: Animal) {
    console.log(animal.makeSound());
}

const dog = new Dog();
makeAnimalSound(dog);  // OK
```

이를 타입 관점으로 보면,

```typescript
// 기본적인 타입 관계
type Animal = {
    name: string;
}

type Dog = {
    name: string;
    bark(): void;
}

const dog: Dog = {
    name: "Buddy",
    bark: () => console.log("Woof!")
};

const animal: Animal = dog;  // 이게 왜 되는 걸까요?
```

이 코드가 동작하는 이유를 LSP로 설명하면 다음과 같다:

1. Dog는 Animal이 할 수 있는 모든 것을 할 수 있음
2. 따라서 Animal이 필요한 곳에 Dog를 넣어도 안전함

이 Animal의 타입에 Dog타입이 할당 가능한 관계를 설명하려면 업캐스팅과 다운 캐스팅의 개념이 적용 된다.
