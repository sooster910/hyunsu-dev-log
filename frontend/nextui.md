---
hidden: true
---

# NextUI 라이브러리

NextUI는 tailwind기반위에서 돌아가기 때문에 tailwind 설치가 필수이다.&#x20;

이 과정에서 여러 설정들이 있는데 어떻게 동작이 되는지 궁금해서 찾아보게 되었다.



1\. **`content` 배열**

`content` 배열은 **Tailwind CSS**가 적용할 파일 경로를 설정하는 부분입니다. Tailwind는 이 파일들에서 사용되는 클래스들을 찾아서, 빌드 시 필요한 스타일을 생성합니다.

```
// Some code

content: [
  "./app/**/*.{js,ts,jsx,tsx,mdx}",
  "./pages/**/*.{js,ts,jsx,tsx,mdx}",
  "./components/**/*.{js,ts,jsx,tsx,mdx}",
  "./src/**/*.{js,ts,jsx,tsx,mdx}",
  "./node_modules/@nextui-org/theme/dist/**/*.{js,ts,jsx,tsx}"
]

```

이 설정은 Tailwind가 **다음 경로들에서 사용되는 클래스**를 모두 감지하도록 합니다:

* `app`, `pages`, `components`, `src` 폴더와 그 하위 디렉토리의 모든 `.js`, `.ts`, `.jsx`, `.tsx`, `.mdx` 파일
* 또한 `@nextui-org/theme` 패키지 내에서 사용되는 Tailwind 클래스를 포함한 CSS도 포함시킵니다. 이렇게 하면 `NextUI`에서 사용하는 CSS도 Tailwind 빌드에 포함되어 스타일이 충돌 없이 통합될 수 있습니다.





#### Tailwind CSS가 실행되는 방식

Tailwind는 다음과 같은 방식으로 동작합니다:

1. **빌드 시, HTML 파일에서 사용된 모든 클래스를 탐지**하고, 이 클래스에 해당하는 CSS 스타일을 생성합니다. 이때 `content` 배열에 지정된 파일들을 검색하여 사용된 클래스를 추출합니다.
2. **사용된 클래스를 기반으로 CSS 파일을 생성**하며, 이 스타일은 Next.js 프로젝트에서 페이지를 렌더링할 때 적용됩니다. 즉, Tailwind는 필요 없는 스타일을 제거하고, 실제로 사용된 스타일만 포함하여 빌드 크기를 최적화합니다.
3. **`@nextui-org/react`와 Tailwind을 결합**하면, NextUI에서 제공하는 UI 컴포넌트를 사용하면서도 Tailwind의 유틸리티 클래스를 자유롭게 사용할 수 있습니다. 예를 들어, NextUI의 버튼 스타일에 Tailwind의 `bg-red-500`, `p-4`, `rounded-lg` 등의 클래스를 추가하여 커스터마이징할 수 있습니다.

####
