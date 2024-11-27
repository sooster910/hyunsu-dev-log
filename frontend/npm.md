---
icon: face-icicles
---

# npm 의존성 충돌부터 마이그레이션, 그외 호환성 트러블슈팅까지

### 문제 발단

개발 환경의 Elastic Beanstalk 배포 과정에서 다음과 같은 NPM 의존성 에러가 발생했다.



<figure><img src="../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>



### 원인 분석

1.  **버전 충돌**

    * 현재 프로젝트: react 17.0.0
    * Next.js 요구사항: react ^17.0.2 이상
    * Sentry 요구사항:  react 16.x || 17.x || 18.x

    <figure><img src="../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>
2.  **Semantic Versioning 이해**

    ```bash
    bashCopy^17.0.2 의미:
    - 최소 버전: 17.0.2
    - 최대 버전: 18.0.0 미만
    - 허용 버전: 17.0.2, 17.0.3, 17.1.0, 17.9.9 등
    ```
3.  **Peer Dependencies**

    * 호스트 패키지(개발중인 패키지)가 특정 버전의 다른 패키지와 호환됨을 명시하는 방법으로서 호환성의 요구 사항이 된다.
    * Peer Dependencies가 필요한 이유는 Peer Dependencies가 없을 경우 중복된 다른 버전의 패키지가 설치 되는데 어플리케이션이 의도대로 동작하지 않는다.&#x20;

    <figure><img src="../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>



### 해결 방안 옵션&#x20;

#### 1. 최소 요구사항 충족하는 방식&#x20;

공통으로 최소한의 요구 버전인 리액트를 17.0.2 로 맞추면 Next 버전은 그대로인 상태로 에러는 해결할 수 있다.

```json
jsonCopy{
  "dependencies": {
    "react": "^17.0.2",
    "react-dom": "^17.0.2",
    "next": "12.3.4"
  }
}
```

하지만 현 시점 Next.js 15가 나온 상황에서 기술 부채가 축적되고 있는 상황이다. \


#### 2. 안정화된 최신 버전으로 맞추는 방식&#x20;

최소한의 버전을 맞춘다는 점에서는 기술부채가 축적되고 있었고, 특히 Next 버전12는 documentation 에서도 이제 더이상 다루지 않는다.  최소가 13.5.7이다.&#x20;

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

12에서 13으로의 업그레이드에는 13은 Page router와 App router를 모두 호환하고 있기 때문에 breaking changes가 많이 없을 거라고 판단되었다.&#x20;

1.  **package.json 수정**

    ```json
    jsonCopy{
      "dependencies": {
        "react": "^17.0.2",
        "react-dom": "^17.0.2",
        "next": "12.3.4",
        "@sentry/nextjs": "^7.91.0"
      }
    }
    ```
2.  **의존성 재설치 및 테스트**

    ```bash
    bashCopy# 의존성 재설치
    npm install

    # 로컬 테스트
    npm run dev

    # 빌드 테스트
    npm run build
    ```



***



### Next13.5 과 UI라이브러리의 Image처리 방식으로 인한 UI 깨짐 현상

위의 순서 까진 좋았다.  이렇게 Next 버전을 업데이트 한 후 Giest 라는 UI라이브러리를 사용 중 이었는데, 어느새 부턴가 메인테이너의 활동이  활발하지 않은걸 확인 하곤 마이그레이션 생각을 하고 있던 찰나였다.

```
// Some code
Warning: GeistImage: Support for defaultProps will be removed from function components in a future major release. Use JavaScript default parameters instead.

```

* GeistUI의  Image는 NextImage기반이 아니여서&#x20;
* 리액트 18+, Next.js13+ 에서는 defaultProps 지원이 중단된다.
* Geist UI가 아직 defaultProps방식을 사용하고 있기에 이런 warning을 내어 알려준다.

점진적으로 alternative한 UI라이브러리로 마이그레이션 하기전까지 당장의 경고를 해결하기 위해 Wrapper Component를 만들었다.

```typescript
// Some code
export const SafeImage = (props: ImageProps) => <Image {...props} />
export const SafeText = (props: TextProps) => <Text {...props} />
```





### NextImage 12 와 13의 Image 속성 호환성 충돌

Next13과의 &#x20;

<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption><p>Next13 Image type</p></figcaption></figure>

next/regacy/image 또한 Safenumber로 더이상 문자열을  받지 않았다.

<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### Lambda 호환 이슈&#x20;

<figure><img src="../.gitbook/assets/image (10).png" alt=""><figcaption></figcaption></figure>

lambda는  definitely typed에서 지원하는 타입정의 파일이 있다.&#x20;

Definitely Typed의 지원 정책을 살펴보면 2년 미만의 TypeScript 버전에 대해서만 패키지를 테스트 한다고 한다.



즉 현재 2024년 11월 시점으로는 5.0까지의 타입정의 파일만 테스트 되므로 이보다 더 오래된 버전일 경우 호환성에 문제가 있을 수 있다.

실무적 입장에서 개발자들은 적어도 2년이내 버전을 사용하는것이 좋다고 . 볼 수 있다.&#x20;

관련 이슈 링크&#x20;

{% @github-files/github-code-block url="https://github.com/DefinitelyTyped/DefinitelyTyped/issues/69932" %}

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption></figcaption></figure>

