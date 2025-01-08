---
description: 마이그레이션 중 리액트
hidden: true
---

# JSX Transform이 Type에도 가져온 영향

사용 하던 라이브러리 제공하는 모든 UI 컴포넌트의 타입 정의가 최신 React JSX 사양과 호환되지 않는데서 문제 였다.&#x20;

<figure><img src="../.gitbook/assets/Screenshot 2024-12-03 at 2.08.50 AM.png" alt=""><figcaption></figcaption></figure>



1. 문제의 대상: `Input` 컴포넌트
2. 문제의 원인: `Input`의 타입인 `InputComponentType`이 React가 예상하는 **JSX 컴포넌트 타입**에 맞지 않음.

&#x20;**왜 `InputComponentType`이 JSX 컴포넌트 타입에 맞지 않는 걸까?**

**리액트는 ReactNode를 반환하길 원하는데**

```
 InputComponentType is not assignable to type
(props: any, deprecatedLegacyContext?: any) => ReactNode

```



실제 Input 컴포넌트의 타입은&#x20;





JSX Transform 이후 TypeScript는 React의 새로운JSX Runtime을 기반으로 동작한다. 그래서 React.createElement를 더이상 직접 호출 하지 않는다.



