---
icon: face-icicles
description: >-
  문제의 발단은 sentry를 추가 하면서 최신버전이다 보니 기존 Nextjs와 의존성 충돌이 일어났고, 이 기회에 12 에서 13 app
  router 사용 버전까지 올리기로 했다.
---

# Next12 -> 13 마이그레이션 호환성 트러블슈팅 기록



### 의존성 충돌&#x20;

sentry를 추가하고 개발 환경의 Elastic Beanstalk 배포 과정에서 다음과 같은 NPM 의존성 에러가  발생했다.

<figure><img src="../.gitbook/assets/image (16) (1).png" alt=""><figcaption></figcaption></figure>

상황을 설명하자면,&#x20;

1.  **버전 충돌**

    * 현재 프로젝트: react 17.0.0
    * Next.js 요구사항: react ^17.0.2 이상
    * Sentry 요구사항:  react 16.x || 17.x || 18.x

    <figure><img src="../.gitbook/assets/image (17) (1).png" alt=""><figcaption></figcaption></figure>
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

    <figure><img src="../.gitbook/assets/image (18) (1).png" alt=""><figcaption></figcaption></figure>

    &#x20;

#### 해결 방안 모색

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

최소한의 버전을 맞춘다는 점에서는 기술부채가 축적되고 있었고, 특히 Next 버전12는 documentation 에서도 이제 더 이상 다루지 않는다.  최소가 13.5.7이다.&#x20;

<figure><img src="../.gitbook/assets/image (13) (1).png" alt=""><figcaption></figcaption></figure>

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

이렇게 Next 버전을 업데이트 한 후 Giest 라는 UI라이브러리를 사용 중 이었는데, 어느새부턴가 메인테이너의 활동이  활발하지 않은걸 확인 하곤 마이그레이션 생각을 하고 있던 찰나였다.

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



<figure><img src="../.gitbook/assets/image (5) (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (6) (1).png" alt=""><figcaption><p>Next13 Image type</p></figcaption></figure>

next/regacy/image 또한 Safenumber로 더이상 문자열을  받지 않았다.

<figure><img src="../.gitbook/assets/image (7) (1).png" alt=""><figcaption></figcaption></figure>

### Lambda 호환 이슈&#x20;

<figure><img src="../.gitbook/assets/image (10) (1).png" alt=""><figcaption></figcaption></figure>

lambda는  definitely typed에서 지원하는 타입정의 파일이 있다.&#x20;

Definitely Typed의 지원 정책을 살펴보면 2년 미만의 TypeScript 버전에 대해서만 패키지를 테스트 한다고 한다.



즉 현재 2024년 11월 시점으로는 5.0까지의 타입정의 파일만 테스트 되므로 이보다 더 오래된 버전일 경우 호환성에 문제가 있을 수 있다. 실무적 입장에서 개발자들은 적어도 2년이내 버전을 사용하는것이 좋다고 볼 수 있다.&#x20;

관련 이슈 링크&#x20;

{% @github-files/github-code-block url="https://github.com/DefinitelyTyped/DefinitelyTyped/issues/69932" %}

<figure><img src="../.gitbook/assets/image (12) (1).png" alt=""><figcaption></figcaption></figure>

{% @github-files/github-code-block url="https://github.com/DefinitelyTyped/DefinitelyTyped/issues/69932" %}



***



### React 18 과 UI라이브러리 타입 호환&#x20;

암묵적으로 허용되던 많은 props들이 이제는 명시적으로 정의되어야 하는데, 이는 React 18에는 더 엄격해졌다.&#x20;

아래는 내가 사용하던 라이브러리의 타입이다.

현재 메인테이너가 활동중이지 않아 더이상의 업데이트는 없다.

이 타입을 바탕으로 Input 컴포넌트를 사용하면 17에서는 나지 않던 타입에러가 18에서는 타입에러로 나타난다.



```typescript
// Some code


import { Props } from "./input-props";
declare type NativeAttrs = Omit<React.InputHTMLAttributes<any>, keyof Props>;
export declare type InputProps = Props & NativeAttrs;
declare const Input: React.ForwardRefExoticComponent<Pick<Props & NativeAttrs & React.RefAttributes<HTMLInputElement> & import("../use-scale").ScaleProps, "min" | "max" | "children" | "form" | "slot" | "style" | "title" | "pattern" | "onClick" | keyof import("../use-scale").ScaleProps | keyof Props | "accept" | "alt" | "autoFocus" | "capture" | "checked" | "crossOrigin" | "enterKeyHint" | "formAction" | "formEncType" | "formMethod" | "formNoValidate" | "formTarget" | "list" | "maxLength" | "minLength" | "multiple" | "name" | "required" | "size" | "src" | "step" | "defaultChecked" | "defaultValue" | "suppressContentEditableWarning" | "suppressHydrationWarning" | "accessKey" | "contentEditable" | "contextMenu" | "dir" | "draggable" | "hidden" | "id" | "lang" | "spellCheck" | "tabIndex" | "translate" | "radioGroup" | "role" | "about" | "datatype" | "inlist" | "prefix" | "property" | "resource" | "typeof" | "vocab" | "autoCapitalize" | "autoCorrect" | "autoSave" | "color" | "itemProp" | "itemScope" | "itemType" | "itemID" | "itemRef" | "results" | "security" | "unselectable" | "inputMode" | "is" | "aria-activedescendant" | "aria-atomic" | "aria-autocomplete" | "aria-busy" | "aria-checked" | "aria-colcount" | "aria-colindex" | "aria-colspan" | "aria-controls" | "aria-current" | "aria-describedby" | "aria-details" | "aria-disabled" | "aria-dropeffect" | "aria-errormessage" | "aria-expanded" | "aria-flowto" | "aria-grabbed" | "aria-haspopup" | "aria-hidden" | "aria-invalid" | "aria-keyshortcuts" | "aria-label" | "aria-labelledby" | "aria-level" | "aria-live" | "aria-modal" | "aria-multiline" | "aria-multiselectable" | "aria-orientation" | "aria-owns" | "aria-placeholder" | "aria-posinset" | "aria-pressed" | "aria-readonly" | "aria-relevant" | "aria-required" | "aria-roledescription" | "aria-rowcount" | "aria-rowindex" | "aria-rowspan" | "aria-selected" | "aria-setsize" | "aria-sort" | "aria-valuemax" | "aria-valuemin" | "aria-valuenow" | "aria-valuetext" | "dangerouslySetInnerHTML" | "onCopy" | "onCopyCapture" | "onCut" | "onCutCapture" | "onPaste" | "onPasteCapture" | "onCompositionEnd" | "onCompositionEndCapture" | "onCompositionStart" | "onCompositionStartCapture" | "onCompositionUpdate" | "onCompositionUpdateCapture" | "onFocusCapture" | "onBlurCapture" | "onChangeCapture" | "onBeforeInput" | "onBeforeInputCapture" | "onInput" | "onInputCapture" | "onReset" | "onResetCapture" | "onSubmit" | "onSubmitCapture" | "onInvalid" | "onInvalidCapture" | "onLoad" | "onLoadCapture" | "onError" | "onErrorCapture" | "onKeyDown" | "onKeyDownCapture" | "onKeyPress" | "onKeyPressCapture" | "onKeyUp" | "onKeyUpCapture" | "onAbort" | "onAbortCapture" | "onCanPlay" | "onCanPlayCapture" | "onCanPlayThrough" | "onCanPlayThroughCapture" | "onDurationChange" | "onDurationChangeCapture" | "onEmptied" | "onEmptiedCapture" | "onEncrypted" | "onEncryptedCapture" | "onEnded" | "onEndedCapture" | "onLoadedData" | "onLoadedDataCapture" | "onLoadedMetadata" | "onLoadedMetadataCapture" | "onLoadStart" | "onLoadStartCapture" | "onPause" | "onPauseCapture" | "onPlay" | "onPlayCapture" | "onPlaying" | "onPlayingCapture" | "onProgress" | "onProgressCapture" | "onRateChange" | "onRateChangeCapture" | "onSeeked" | "onSeekedCapture" | "onSeeking" | "onSeekingCapture" | "onStalled" | "onStalledCapture" | "onSuspend" | "onSuspendCapture" | "onTimeUpdate" | "onTimeUpdateCapture" | "onVolumeChange" | "onVolumeChangeCapture" | "onWaiting" | "onWaitingCapture" | "onAuxClick" | "onAuxClickCapture" | "onClickCapture" | "onContextMenu" | "onContextMenuCapture" | "onDoubleClick" | "onDoubleClickCapture" | "onDrag" | "onDragCapture" | "onDragEnd" | "onDragEndCapture" | "onDragEnter" | "onDragEnterCapture" | "onDragExit" | "onDragExitCapture" | "onDragLeave" | "onDragLeaveCapture" | "onDragOver" | "onDragOverCapture" | "onDragStart" | "onDragStartCapture" | "onDrop" | "onDropCapture" | "onMouseDown" | "onMouseDownCapture" | "onMouseEnter" | "onMouseLeave" | "onMouseMove" | "onMouseMoveCapture" | "onMouseOut" | "onMouseOutCapture" | "onMouseOver" | "onMouseOverCapture" | "onMouseUp" | "onMouseUpCapture" | "onSelect" | "onSelectCapture" | "onTouchCancel" | "onTouchCancelCapture" | "onTouchEnd" | "onTouchEndCapture" | "onTouchMove" | "onTouchMoveCapture" | "onTouchStart" | "onTouchStartCapture" | "onPointerDown" | "onPointerDownCapture" | "onPointerMove" | "onPointerMoveCapture" | "onPointerUp" | "onPointerUpCapture" | "onPointerCancel" | "onPointerCancelCapture" | "onPointerEnter" | "onPointerEnterCapture" | "onPointerLeave" | "onPointerLeaveCapture" | "onPointerOver" | "onPointerOverCapture" | "onPointerOut" | "onPointerOutCapture" | "onGotPointerCapture" | "onGotPointerCaptureCapture" | "onLostPointerCapture" | "onLostPointerCaptureCapture" | "onScroll" | "onScrollCapture" | "onWheel" | "onWheelCapture" | "onAnimationStart" | "onAnimationStartCapture" | "onAnimationEnd" | "onAnimationEndCapture" | "onAnimationIteration" | "onAnimationIterationCapture" | "onTransitionEnd" | "onTransitionEndCapture" | "key"> & React.RefAttributes<HTMLInputElement>>;
export default Input;

```



React 17에서는 `Pick`으로 선택된 속성이 **required**로 정의되어 있었더라도, TypeScript와 React의 타입 시스템이 이를 엄격하게 확인하지 않았습니다. 이는 React 17의 타입 정의와 당시 TypeScript 버전의 동작 방식 때문입니다. 자세히 살펴보겠습니다.



이전에는 `React.RefAttributes`와 같은 속성이 암묵적으로 추가되었지만, React 18에서는 모든 속성을 명시적으로 포함하거나 정의해야 합니다.`Pick` 사용 시, 선택되지 않은 속성이 누락으로 간주된다.



