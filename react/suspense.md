---
hidden: true
---

# Suspense

먼저 Suspense 가 리액트에서 어떻게 지원해 왔는지 정리 해보자.

사실 Supsense에 관심을 갖게된건 얼마 되지 않았다. Suspense라는 기능은 알고 있었지만 최적화시 lazy할 때 정도 가끔 사용하는 것 외에 크게 사용할 기회가 없었고, 잘 알지 못하니 그만큼 우선순위에서  높게 생각 못했던건 사실이다.&#x20;



#### 📅 **React 16.6 (2018년)**

* `Suspense`는 `React.lazy()`와 함께 **컴포넌트 로딩**에 사용되었습니다. 데이터를 가져오는 것에는 사용되지 않았습니다.

#### 🚀 **React 18 (2022년)**

* **React 18**에서 데이터 페칭을 위한 `Suspense`가 지원되었고, 이를 통해 서버 컴포넌트와 클라이언트 컴포넌트 간의 데이터 로딩을 동기화하는 방식으로 더욱 발전했다.
* React 18에서 `Suspense`와 **`useDeferredValue`** 등이 데이터 페칭과 관련된 기능을 지원하며, 이를 통해 데이터가 준비될 때까지 로딩 상태를 표시하거나 지연시키는 방법을 제공하게 되었다.

#### ⚡ **Next.js 13에서의 변화**

* **Next.js 13**부터는 React 18의 기능을 활용하여 **서버 컴포넌트**와 **클라이언트 컴포넌트** 간의 상호작용이 가능해졌다.
* Next.js는 기본적으로 `Suspense`를 사용해 데이터를 처리할 수 있게 되었으며, 서버 사이드에서 데이터를 패칭하고, 클라이언트에서 렌더링할 수 있는 구조를 제공합니다









<figure><img src="../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

\
\


<figure><img src="../.gitbook/assets/image (37).png" alt=""><figcaption></figcaption></figure>



여기에서 좀 더 개선할 순 없을까?



<figure><img src="../.gitbook/assets/image (38).png" alt=""><figcaption></figcaption></figure>
